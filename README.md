Recursion schemes
=========================

This repository contains documentation, samples and exercises for the Recursion Schemes workshop.

**Running**  
Load this project in sbt and just launch `run`, then select the exercise you'd like to execute.

## Introduction

Recursive structures are a common pattern and most developers have worked with such data at least a few times. 

### Trees
**Nodes** and **leafs**

```
           root-node
           /   \    
  child-node   leaf
  /     \
leaf    leaf  
```

### Json

TODO

### List

TODO

### ASTs

```sql
SELECT * FROM users u WHERE u.age > 25 AND UPPER(u.name) LIKE "J%"  
```

```
FILTER:
- source:  
    SELECT  
        -- selection: *  
        -- from: users  
- criteria:  
    AND  
        - GT(u.age, 25)  
        - LIKE(  
            UPPER(  
                u.name)  
            J%)      
```

### What you can do with recursive data
- Print
- Translate into another structure
- Enrich
- Optimize


## Manual recursion 

```scala
sealed trait Expr

case class IntValue(v: Int)           extends Expr
case class DecValue(v: Double)        extends Expr
case class Sum(a: Expr, b: Expr)      extends Expr
case class Multiply(a: Expr, b: Expr) extends Expr
case class Divide(a: Expr, b: Expr)   extends Expr
  
def eval(e: Expr): Double =
e match {
  case IntValue(v)      => v.toDouble
  case DecValue(v)      => v
  case Sum(e1, e2)      => eval(e1) + eval(e2)
  case Multiply(e1, e2) => eval(e1) * eval(e2)
  case Divide(e1, e2)   => eval(e1) / eval(e2)
}
```

## Fixed point datatypes

### Making an ADT polymorphic

Ideally we'd love to have something more elegant.
We are looking for a tool which takes:
- A recursive expression of type Expr, 
- a function which evaluates `Expr => Double`  
  For example `case Sum(d1, d2) => d1 + d2`
Such tool evaluates whole expression to a `Double`

Types like `Sum(a: Expr, b: Expr)` force us to deal only with Exprs. 
Ideally we'd like to have our eval definition to look like:
```scala
// does not compile, but it's only an illustration of a direction
def eval(e: Expr): Double = 
e match {
  case Sum(dbl1: Double, dbl2: Double) => dbl1 + dbl2 // etc
} 
``` 

Let's make our expression **polymorphic**.

```scala
sealed trait Expr[A]

case class IntValue[A](v: Int)           extends Expr[A]
case class DecValue[A](v: Double)        extends Expr[A]
case class Sum[A](a: A, b: A)            extends Expr[A]
case class Multiply[A](a: A, b: A)       extends Expr[A]
case class Divide[A](a: A, b: A)         extends Expr[A]
```

That's much better, because this allows us express our evaluations as:
```scala
def evalToDouble(exp: Expr[Double]): Double = exp match {
  case IntValue(v) => v.toDouble
  case DecValue(v) => v
  case Sum(d1, d2) => d1 + d2
  case Multiply(d1, d2) => d1 * d2
  case Divide(d1, d2) => d1 / d2
} 
```
Such evaluation is what we aim for, because it doesn't look like
recursion. It looks more like a set of rules, which we can **apply** to
a recursive structure with some blackbox tool which will recursively
build the result.

But let's stop here for a while, because polymorphic expressions
come with a cost... First, consider a trivial expression:  
`val intVal = IntValue[Unit](10) // Expr[Unit]`

What about more complex expressions?

```scala
  val sumExp: Expr[Expr[Unit]] =
    Sum(
      IntValue[Unit](10), // Expr[Unit]
      IntValue[Unit](5)
    )
```

### Fixing nested Exprs

how to deal with types like `Expr[Expr[Expr[A]]]`?
Let's wrap in:  

```scala
case class Fix[F[_]](unFix: F[Fix[F]])
  
val fixedIntExpr: Fix[Expr] = Fix(IntValue[Fix[Expr]](10))
```

The `Fix` type allows us to represent any `Expr[Expr[Expr....[A]]]` as `Fix[Expr]`

Wait, why did we need this`Fix` thing?

### A step back

We wanted to use evaluation definition which doesn't look like recursion. 

We are looking for a tool which takes:
- A recursive expression of type Expr, 
- a function which evaluates a single simple `Expr => Double`  
  For example `case Sum(d1, d2) => d1 + d2`

To be able to express such rules, we needed to go from `Expr` to `Expr[A]`.
To avoid issues with nested types, we introduced `Fix[Expr]`

### Putting it all together

Once we have:
- A polymorphic recursive structure based on `Expr[A]`
- An evaluation recipe expressed as a set of rules for  each sub-type (`Expr[B] => B`)
- A `Fix[F[_]]` wrapper

We can now use a tool to put this all together. Such tool is called...

## Catamorphism

### Scheme

A generic **foldRight** for data stuctures. In case of recursive data,
this means **folding bottom-up**:

```scala
  val division =
    Divide(DecValue(5.2), Sum(IntValue(10), IntValue(5)))
```

```
            Divide                             Divide
           /    \                              /    \
DecValue(5.2)   Sum            -->   DecValue(5.2)  Sum
                / \                                 / \
     IntValue(10)  IntValue(5)                   10.0 5.0
```

```
            Divide                             Divide
           /    \                              /    \
DecValue(5.2)   Sum            -->            5.2  15.0
                / \                                 
             10.0  5.0                  
```

```
            Divide             -->            5.2 / 15.0                             
           /    \                              
         5.2   15.0            
```

Going **bottom-up**, we use our set of rules on leafs, then we build
higher nodes **basing** on lower nodes. Catamorphism is a **generic** tool,
so you don't have to implement it!

### Matryoshka and cata

The Matryoshka library does catamorphism for you:

```scala

val recursiveExpr: Fix[Expr] = ??? // your tree

def evalToDouble(expr: Expr[Double]): Double

// the magic call
rcursiveExpression.cata(evalToDouble) // returns Double
```

The `.cata()` call runs the whole folding process and constructs 
the final `Double` value for you, provided just a set of rules for
indiviual node types.

### Expression functor

Matryoshka's `.cata()` is a blackbox, but it has one more requirement.
It's mechanism assumes that a `Functor` instance is available for your datatype.

This means that you must provide a recipe for how to **map** `Expr[A]` to `Expr[B]`
having a function `f: A => B`.
For example to transform `Sum[A](a1: A, a2: A)` into `Sum[B](b1: B, b2: B)` you need
to do `Sum(f(a1), f(a2))`. Such recipe has to be provided for all
possible cases of `Expr`.

```scala
import scalaz.Functor  

implicit val exprFunctor: Functor[Expr] = new Functor[Expr] {
  override def map[A, B](expr: Expr[A])(f: A => B): Expr[B] = expr match {
    case IntVal(v) => IntVal[B](v)
    case Sum(a1, a2) => Sum(f(a1), f(a2))
    case ... // etc.
  }      
}
```

This is finally all what we need! Here's a summary of our ingredients:

1. A recursive structure `Expr[A]`
2. A Functor for this type
3. A set or evaulation rules for **individual cases**
4. `Fix[[_]]`
5. catamorphism

4 & 5 are provided by Matryoshka.

### Algebra

Our evaluation function (point 3.) is called an **Algebra**. From Matryoshka:

```scala
type Algebra[F[_], A] = F[A] => A
    
def evalToDouble(expr: Expr[Double]): Double
  
val evalToDouble: Algebra[Expr, Double]
```

### Some syntax sugar to work with Fix

Remember this?
```scala
Fix(Sum(Fix(IntValue[Fix[Expr]](10)), Fix(IntValue[Fix[Expr]](5))))
```

There`s some syntax sugar to help:
```scala
Sum(IntValue[Fix[Expr]](10).embed, IntValue[Fix[Expr]](5).embed).embed
```

Handy especially for larger expressions (see exercise 03). To unpack from `Fix`, you can
use `unFix`:

```scala
val fixedSum: Fix[Expr] = 
  Sum(
    IntValue[Fix[Expr]](10).embed, 
    IntValue[Fix[Expr]](5).embed
  ).embed
  
fixedSum.unFix match {
  case Sum(...) =>
}
``` 

`unFix` is fine, but instead use `.project` which does the same, but is a more general
function which work on other recursive wrappers. Sorry for the spoiler, but `Fix` is not
the only one around, and you don't want to get tied directly to it! 

### Transforming recursive expressions

Let's say we have an `Expr` and we want to optimize it to express `Multiply(x, x)` as `Square(x)`.
We'd like to have a tool which walks our tree and **maps** a given `Expr` to another `Expr`

// TODO check Pawel's talk

```
           Divide                                     Divide
           /    \                                     /    \
DecValue(5.2)   Multiply         -->       DecValue(5.2) Square(10)
                  / \                               
       IntValue(10)  IntValue(10)                   
```

We are looking for a function like:

```scala
  def mapNode(t: Fix[Expr])(f: Fix[Expr] => Fix[Expr])
  
  def optimize(expr: Fix[Expr]): Fix[Expr] = expr.project match {
    case Multiply(a, b) if (a == b) => Square(a)      
    case other => other
  }
  
  val optimizedExpr = mapNode(exprTree)(optimize)
```

Matroysha offers such a tool, and it's called `transCataT`:

```scala
  val optimizedExpr: Fix[Expr] = exprTee.transCataT(optimize) 
```

### cataM

Sometimes our evaluation produces a wrapped value. Suppose we want to evaluate the
expression to a Double, but handle division by zero by wrapping the result in an `Either`.
Here's our new evaluation:

```scala
def evalToDouble(exp: Expr[Double]): \/[String, Double] = exp match {
  case IntValue(v) => v.toDouble.right
  case DecValue(v) => v.right
  case Sum(d1, d2) => (d1 + d2).right
  case Multiply(d1, d2) => (d1 * d2).right
  case Divide(d1, d2) => 
  if (d2 == 0) 
    (d1 / d2).right
  else
    "Division by zero!".left
  case Square(d) => (d * d).right
} 
```

We can't just use `cata`, because `cata` works with `Algebra`.  

```scala
type Algebra[F[_], A] = F[A] => A

// F[A] => M[A], where M[A] means Either[String, A]
def evalToDouble(exp: Expr[Double]): Either[String, Double] 
```
`Algebra` is a function `type Algebra[F[_], A] = F[A] => A`, while our new evaluation is
of type `F[A] => M[A]`. If our evaluation has such signature, we can use **cataM**:

```scala
val correctExpr: Fix[Expr] =
  Sum(
    DecValue[Fix[Expr]](5.2).embed,
    Divide(
      DecValue[Fix[Expr]](3.0).embed,
      DecValue[Fix[Expr]](3.0).embed
    ).embed
  ).embed

val incorrectExpr: Fix[Expr] =
  Sum(
    DecValue[Fix[Expr]](5.2).embed,
    Divide(
      DecValue[Fix[Expr]](3.0).embed,
      DecValue[Fix[Expr]](0.0).embed // !!!!!!!!
    ).embed
  ).embed 
  
correctExpr.cataM(evalToDouble) // Right(6.2)
incorrectExpr.cataM(evalToDouble) // Left("Division by zero!")
```

However, there's one more requirement. Our `Functor[Expr]` is not enough, Matryoshka needs
a `Traverse[Expr]`.  
Advice: Instead of writing a `Functor` for your recursive data type, write a `Traverse`. It is
also a `Functor`, it's pretty much the same amount of work, and it may become useful in case
you need `cataM`.

## Anamorphism

As we learned, cata is a bottom-up folding of a structure. Anamorphism works in the opposite direction
and allows to unfold a structure. For this we need the dual of `Algebra` - `Coalgebra`:
```scala
type Coalgebra[F[_], A] = A => F[A]
``` 

Such morphism is called an unfold, because it takes an object and recursively builds up a structure
basing on it.
```scala

// Int => Expr[Int]
val toBinary: Coalgebra[Expr, Int] = (n: Int) =>
n match {
  case 0 => IntValue(0)
  case 1 => IntValue(1)
  case 2 => IntValue(2)
  case _ if n % 2 == 0 => Multiply(2, n / 2)
  case _ => Sum(1, n - 1)
}

val toText: Algebra[Expr, String] = {
case IntValue(v)    => v.toString
case Sum(a, b)      => s"($a + $b)"
case Multiply(a, b) => s"($a * $b)"
}

// unfold with anamorphism
val expr = 31.ana.apply[Fix[Expr]](toBinary)
// and now fold with catamorphism 
val binAsStr = expr.cata(toText) // (1 + (2 * (1 + (2 * (1 + (2 * (1 + 2)))))))
```

A composition of ana+cata is called **hylomorphism**

```scala
val binAsStr = 31.hylo(toText, toBinary)
```

## Paramorphism

Also a fold, very similar to catamorphism. However, it adds one more extra feature - 
with paramorphism, you not only build current state basing on evaluation of previous states,
buy you also have access to these states. Confusing? Here's an example:  
Let's say we want to use a special algorithm for a specific case: 

We want to print `(3 + -5)` as `(5 - 3)` for better readability. 

With cata, we don't have enough information except Strings:

```scala
case Sum(left: String, right: String) => ??? // what was left and right before their evaluation?
```

Parsing Strings here doesn't seem like a good idea. With para, we can use a richer kind of algebra:

```scala
case Sum((leftSrc, leftStr), (rightSrc, rightStr)) =>
  leftSrc.project match {
    case IntValue(a) =>
      rightSrc.project match {
        case IntValue(b) if a > 0 && b < 0 => s"($a - ${-b})"
        case IntValue(b) if b > 0 && a < 0 => s"($b - ${-a})"
        case _                             => s"$leftStr + $rightStr"
      }
    case _ => s"$leftStr + $rightStr"
  }
```

This is a different kind of Algebra: `Expr[(Fix[Expr], String)] => String`, so it's
`F[(T, A)] => A`, where `T` means `Fix[Expr]` for this particular case.
To be more precise, it's
```scala
type GAlgebra[W[_], F[_], A] = F[W[A]] => A
```
Where in our case `F[W[A]]` is `Expr[Tuple2[T, A]]`.   
Paramorphism is useful when we need to know more about node's childrens' structure in order to fully
evaulate that node.
Paramorphisms are generalized folds with access to the input argument corresponding 
to the most recent state of the computation.
However, para is limited, because we only know the source **structure**. If you need more,
meet histomorphism.

## Histomorphism

Histomporphism is a fold which also provides something akin to a "pair". While in paramorphism for each node we
could access pairs of (child_evaluated + child_sourceExpression), in histomorphism we work with
(child_evaluated + child_tree). This child_tree can be traversed further down. To represent this whole pair, we 
use `Cofree`.

### Cofree

Cofree is a very broad concept. It's actually a "comonad" with quite a few interesting attributes and applications.
For the sake of this course, let's consider `Cofree[S, A]` to be a **pair** of:

1. a value of type A
2. A recursive expression S which can consist of deeper elements of type Cofree.

Cofree is often used to construct labelled expression. Each node in an expression gets an extra label (tag).
For a simple DSL:
```scala
sealed trait Expr[A]

case class IntValue[A](v: Int)     extends Expr[A]
case class DecValue[A](v: Double)  extends Expr[A]
case class Sum[A](a: A, b: A)      extends Expr[A]
case class Square[A](a: A)         extends Expr[A]
``` 

we can label any `Expr` with an `ExprType`:

```scala
sealed trait ExprType
case object IntExpr extends ExprType
case object DecExpr extends ExprType
```

Our rules can be simple: a sum of integers is an integer. A sum of (int + dec) is a decimal. A square of a type 
has the same type. A **tagged** expression is of type `Cofree[Expr, ExprType]`. Such object cosists of:

1. A `head: ExprType` which is the tag (label) placed on the node.
2. A `tail: Expr[Cofree[Expr, ExprType]]` which is the expression itself (`Expr[A]`) where `A` is a deeper level of `Cofree`.

We can recursively apply such labelling with catamorphism, using an `Algebra[Expr, Cofree[Expr, ExprType]]`

```scala
val inferType: Algebra[Expr, Cofree[Expr, ExprType]] = {
  case IntValue(v) => Cofree.apply(IntExpr, IntValue(v))
  case DecValue(v) => Cofree.apply(DecExpr, DecValue(v))
  case Sum(a, b) => ??? // a: Cofree[Expr, ExprType]]
}
```  
