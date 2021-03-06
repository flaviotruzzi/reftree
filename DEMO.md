## Unzipping Immutability

This page contains the materials for my talk “Unzipping Immutability”.
Here are some past and future presentations:

* [LX Scala 2016](http://www.lxscala.com/schedule/#session-2) ([video](https://vimeo.com/162214356)).
* [Pixels Camp 2016](https://github.com/PixelsCamp/talks/blob/master/unzipping-immutability_nick-stanchenko.md).
* [Scala By The Bay 2016](http://sched.co/7iTv).

You can use this page in two ways:

* as a reference/refresher on the concepts covered in the talk;
* as an interactive playground where you can try the same commands I presented.

Here is an overview:

* [Immutable data structures (wip)](#immutable-data-structures)
* [Lenses](#lenses)
* [Zippers](#zippers)
* [Useful resources](#useful-resources)

Throughout this page we will assume the following
declarations (each section might add its own):

```scala
import reftree._
import reftree.demo.Data._
import scala.collection.immutable._
import java.nio.file.Paths

val diagram = Diagram(
  defaultOptions = Diagram.Options(density = 100),
  defaultDirectory = Paths.get("images")
)
```

To start an interactive session, just run

```
$ sbt demo
@ render(List(1, 2, 3))
```

and open `diagram.png` in your favorite image viewer (hopefully one that
reloads images automatically on file change). You will also need to have
[GraphViz](http://www.graphviz.org/) installed. *The interactive session
already has all the necessary imports in scope.*

### Immutable data structures

This section is not ready yet, but in the meantime you can find
a few interesting visualizations on the [main page](https://github.com/stanch/reftree).


### Lenses

Updating immutable data structures can be tricky.
For case classes Scala gives us the `copy` method:

```scala
case class Employee(
  name: String,
  salary: Long
)
```

```scala
scala> employee
res1: reftree.demo.Data.Employee = Employee(Michael,4000)

scala> val raisedEmployee = employee.copy(salary = employee.salary + 10)
raisedEmployee: reftree.demo.Data.Employee = Employee(Michael,4010)
```

However once composition comes into play, the resulting nested immutable data structures
would require a lot of `copy` calls:

```scala
case class Employee(
  name: String,
  salary: Long
)

case class Startup(
  name: String,
  founder: Employee,
  team: List[Employee]
)
```

```scala
scala> startup
res2: reftree.demo.Data.Startup = Startup(Acme,Employee(Michael,4000),List(Employee(Adam,2100), Employee(Bella,2100), Employee(Chad,1980), Employee(Delia,1850)))

scala> val raisedFounder = startup.copy(
     |   founder = startup.founder.copy(
     |     salary = startup.founder.salary + 10
     |   )
     | )
raisedFounder: reftree.demo.Data.Startup = Startup(Acme,Employee(Michael,4010),List(Employee(Adam,2100), Employee(Bella,2100), Employee(Chad,1980), Employee(Delia,1850)))
```

```scala
// extra declarations for this section
import reftree.contrib.SimplifiedInstances.list
import reftree.contrib.LensInstances._

val diagram = Diagram(
  defaultOptions = Diagram.Options(density = 100),
  defaultDirectory = Paths.get("images", "lenses")
)
```

```scala
diagram.render("startup")(startup, raisedFounder)
```

<p align="center"><img src="images/lenses/startup.png" width="100%" /></p>

Ouch!

A common solution to this problem is a “lens”.
In the simplest case a lens is a pair of functions to get and set a value of type `B` inside a value of type `A`.
It’s called a lens because it focuses on some part of the data and allows to update it.
For example, here is a lens that focuses on an employee’s salary
(using the excellent [Monocle library](https://github.com/julien-truffaut/Monocle)):

```scala
scala> import monocle.macros.GenLens
import monocle.macros.GenLens

scala> val salaryLens = GenLens[Employee](_.salary)
warning: there was one feature warning; re-run with -feature for details
salaryLens: monocle.Lens[reftree.demo.Data.Employee,Long] = $anon$1@1d3d66c1

scala> salaryLens.get(startup.founder)
res6: Long = 4000

scala> salaryLens.modify(s => s + 10)(startup.founder)
res7: reftree.demo.Data.Employee = Employee(Michael,4010)
```

```scala
diagram.render("salaryLens")(salaryLens → startup.founder)
```

<p align="center"><img src="images/lenses/salaryLens.png" width="40%" /></p>

We can also define a lens that focuses on the startup’s founder:

```scala
scala> val founderLens = GenLens[Startup](_.founder)
warning: there was one feature warning; re-run with -feature for details
founderLens: monocle.Lens[reftree.demo.Data.Startup,reftree.demo.Data.Employee] = $anon$1@ebcbb86

scala> founderLens.get(startup)
res9: reftree.demo.Data.Employee = Employee(Michael,4000)
```

```scala
diagram.render("founderLens")(founderLens → startup)
```

<p align="center"><img src="images/lenses/founderLens.png" width="100%" /></p>

It’s not apparent yet how this would help, but the trick is that lenses can be composed:

```scala
scala> val founderSalaryLens = founderLens composeLens salaryLens
founderSalaryLens: monocle.PLens[reftree.demo.Data.Startup,reftree.demo.Data.Startup,Long,Long] = monocle.PLens$$anon$1@1c3ba0cd

scala> founderSalaryLens.get(startup)
res11: Long = 4000

scala> founderSalaryLens.modify(s => s + 10)(startup)
res12: reftree.demo.Data.Startup = Startup(Acme,Employee(Michael,4010),List(Employee(Adam,2100), Employee(Bella,2100), Employee(Chad,1980), Employee(Delia,1850)))
```

```scala
diagram.render("founderSalaryLens")(founderSalaryLens → startup)
```

<p align="center"><img src="images/lenses/founderSalaryLens.png" width="100%" /></p>

One interesting thing is that lenses can focus on anything, not just direct attributes of the data.
Here is a traversal — a more generic kind of lens — that focuses on all vowels in a string:

```scala
diagram.render("vowelTraversal")(vowelTraversal → "example")
```

<p align="center"><img src="images/lenses/vowelTraversal.png" width="40%" /></p>

We can use it to give our founder a funny name:

```scala
scala> val employeeNameLens = GenLens[Employee](_.name)
warning: there was one feature warning; re-run with -feature for details
employeeNameLens: monocle.Lens[reftree.demo.Data.Employee,String] = $anon$1@7ab94aa1

scala> val founderVowelLens = founderLens composeLens employeeNameLens composeTraversal vowelTraversal
founderVowelLens: monocle.PTraversal[reftree.demo.Data.Startup,reftree.demo.Data.Startup,Char,Char] = monocle.PTraversal$$anon$2@7ba8ae9a

scala> founderVowelLens.modify(v => v.toUpper)(startup)
res15: reftree.demo.Data.Startup = Startup(Acme,Employee(MIchAEl,4000),List(Employee(Adam,2100), Employee(Bella,2100), Employee(Chad,1980), Employee(Delia,1850)))
```

```scala
diagram.render("founderVowelLens")(founderVowelLens → startup)
```

<p align="center"><img src="images/lenses/founderVowelLens.png" width="100%" /></p>

So far we have replaced the `copy` boilerplate with a number of lens declarations.
However most of the time our goal is just to update data.

In Scala there is a great library called [quicklens](https://github.com/adamw/quicklens)
that allows to do exactly that, creating all the necessary lenses under the hood:

```scala
scala> import com.softwaremill.quicklens._
import com.softwaremill.quicklens._

scala> val raisedCeo = startup.modify(_.founder.salary).using(s => s + 10)
raisedCeo: reftree.demo.Data.Startup = Startup(Acme,Employee(Michael,4010),List(Employee(Adam,2100), Employee(Bella,2100), Employee(Chad,1980), Employee(Delia,1850)))
```

You might think this is approaching the syntax for updating mutable data,
but actually we have already surpassed it, since lenses are much more flexible:


```scala
scala> val raisedEveryone = startup.modifyAll(_.founder.salary, _.team.each.salary).using(s => s + 10)
raisedEveryone: reftree.demo.Data.Startup = Startup(Acme,Employee(Michael,4010),List(Employee(Adam,2110), Employee(Bella,2110), Employee(Chad,1990), Employee(Delia,1860)))
```


### Zippers

In our domain models we are often faced with recursive data structures.
Consider this example:

```scala
case class Employee(
  name: String,
  salary: Long
)

case class Hierarchy(
  employee: Employee,
  team: List[Hierarchy]
)

case class Company(
  name: String,
  hierarchy: Hierarchy
)
```

The `Hierarchy` class refers to itself.
Let’s grab a company object and display its hierarchy as a tree:

```scala
// extra declarations for this section
import zipper._
import reftree.contrib.SimplifiedInstances.option

val diagram = Diagram(
  defaultOptions = Diagram.Options(density = 100),
  defaultDirectory = Paths.get("images", "zippers")
)
```

```scala
diagram.render("company")(company.hierarchy)
```

<p align="center"><img src="images/zippers/company.png" width="100%" /></p>

What if we want to navigate through this tree and modify it along the way?
We can use [lenses](#lenses), but the recursive nature of the tree allows for a better solution.

This solution is called a “Zipper”, and was introduced by Gérard Huet in 1997.
It consists of a “cursor” pointing to a location anywhere in the tree — “current focus”.
The cursor can be moved freely with operations like `moveDownLeft`, `moveRight`, `moveUp`, etc.
Current focus can be updated, deleted, or new nodes can be inserted to its left or right.
Zippers are immutable, and every operation returns a new Zipper.
All the changes made to the tree can be committed, yielding a new modified version of the original tree.

Here is how we would insert a new employee into the hierarchy:

```scala
val updatedHierarchy = Zipper(company.hierarchy).moveDownRight.moveDownRight.insertRight(newHire).commit
```

```scala
diagram.render("updatedHierarchy")(company.hierarchy, updatedHierarchy)
```

<p align="center"><img src="images/zippers/updatedHierarchy.png" width="100%" /></p>

My [zipper library](https://github.com/stanch/zipper#zipper--an-implementation-of-huets-zipper)
provides a few useful movements and operations.

Let’s consider a simpler recursive data structure:

```scala
case class Tree(x: Int, c: List[Tree] = List.empty)
```

and a simple tree:

```scala
val tree1 = Tree(
  1, List(
    Tree(2),
    Tree(3),
    Tree(4),
    Tree(5)
  )
)
```

```scala
diagram.render("tree1")(tree1)
```

<p align="center"><img src="images/zippers/tree1.png" width="50%" /></p>

When we wrap a Zipper around this tree, it does not look very interesting yet:

```scala
val zipper1 = Zipper(tree1)
```

```scala
diagram.render("zipper1")(tree1, zipper1)
```

<p align="center"><img src="images/zippers/zipper1.png" width="50%" /></p>

We can see that it just points to the original tree and has some other empty fields.
More specifically, a Zipper consists of four pointers:

```scala
case class Zipper[A](
  left: List[A],           // left siblings of the focus
  focus: A,                // the current focus
  right: List[A],          // right siblings of the focus
  top: Option[Zipper[A]]   // the parent zipper
)
```

In this case the focus is the root of the tree, which has no siblings,
and the parent zipper does not exist, since we are at the top level.

One thing we can do right away is modify the focus:

```scala
val zipper2 = zipper1.update(focus ⇒ focus.copy(x = focus.x + 99))
```

```scala
diagram.render("zipper2")(tree1, zipper1, zipper2)
```

<p align="center"><img src="images/zippers/zipper2.png" width="50%" /></p>

We just created a new tree! To obtain it, we have to commit the changes:

```scala
val tree2 = zipper2.commit
```

```scala
diagram.render("tree2")(tree1, tree2)
```

<p align="center"><img src="images/zippers/tree2.png" width="50%" /></p>

If you were following closely,
you would notice that nothing spectacular happened yet:
we could’ve easily obtained the same result by modifying the tree directly:

```scala
val tree2b = tree1.copy(x = tree1.x + 99)

assert(tree2b == tree2)
```

The power of Zipper becomes apparent when we go one or more levels deep.
To move down the tree, we “unzip” it, separating the child nodes into
the focused node and its left and right siblings:

```scala
val zipper2 = zipper1.moveDownLeft
```

```scala
diagram.render("zipper1+2")(zipper1, zipper2)
```

<p align="center"><img src="images/zippers/zipper1+2.png" width="50%" /></p>

The new Zipper links to the old one,
which will allow us to return to the root of the tree when we are done applying changes.
This link however prevents us from seeing the picture clearly.
Let’s elide the parent field:

```scala
import reftree.contrib.SimplifiedInstances.zipper
```

```scala
diagram.render("zipper2b")(zipper2)
```

<p align="center"><img src="images/zippers/zipper2b.png" width="50%" /></p>

Great! We have `2` in focus and `3, 4, 5` as right siblings. What happens if we move right a bit?

```scala
val zipper3 = zipper2.moveRightBy(2)
```

```scala
diagram.render("zipper3")(zipper3)
```

<p align="center"><img src="images/zippers/zipper3.png" width="50%" /></p>

This is interesting! Notice that the left siblings are “inverted”.
This allows to move left and right in constant time, because the sibling
adjacent to the focus is always at the head of the list.

This also allows us to insert new siblings easily:

```scala
val zipper4 = zipper3.insertLeft(Tree(34))
```

```scala
diagram.render("zipper4")(zipper4)
```

<p align="center"><img src="images/zippers/zipper4.png" width="50%" /></p>

And, as you might know, we can delete nodes and update the focus:

```scala
val zipper5 = zipper4.deleteAndMoveRight.set(Tree(45))
```

```scala
diagram.render("zipper5")(zipper5)
```

<p align="center"><img src="images/zippers/zipper5.png" width="50%" /></p>

Here is an animation of the focus moving between the sibling nodes:

```scala
diagram.renderAnimation("sideways", tweakOptions = _.copy(onionSkinLayers = 0))(
  Utils.iterate(zipper2)(
    _.moveRight, _.moveRight, _.moveRight,
    _.moveLeft, _.moveLeft, _.moveLeft
  )
)
```

<p align="center"><img src="images/zippers/sideways.gif" width="60%" /></p>

Finally, when we move up, the siblings at the current level are “zipped”
together and their parent node is updated:

```scala
val zipper6 = zipper5.moveUp
```

```scala
diagram.render("zipper6")(zipper6)
```

<p align="center"><img src="images/zippers/zipper6.png" width="50%" /></p>

You can probably guess by now that `.commit` is a shorthand for going
all the way up (applying all the changes) and returning the focus:

```scala
val tree3a = zipper5.moveUp.focus
val tree3b = zipper5.commit

assert(tree3a == tree3b)
```


### Useful resources

#### Books, papers and talks

* [Purely functional data structures](http://www.amazon.com/Purely-Functional-Structures-Chris-Okasaki/dp/0521663504) by Chris Okasaki,
  and/or [his PhD thesis](https://www.cs.cmu.edu/~rwh/theses/okasaki.pdf) — *the* introduction to immutable data structures
* [What’s new in purely functional data structures since Okasaki](http://cstheory.stackexchange.com/a/1550) — an excellent StackExchange answer
  with pointers for further reading
* [Extreme cleverness](https://www.youtube.com/watch?v=pNhBQJN44YQ) by Daniel Spiewak — a superb talk
  covering several immutable data structures (implemented [here](https://github.com/djspiewak/extreme-cleverness))
* [Finger Trees](http://www.cs.ox.ac.uk/ralf.hinze/publications/FingerTrees.pdf) and
  [1-2 Brother Trees](http://www.cs.ox.ac.uk/ralf.hinze/publications/Brother12.pdf) described by Hinze and Paterson
* [Huet’s original Zipper paper](https://www.st.cs.uni-saarland.de/edu/seminare/2005/advanced-fp/docs/huet-zipper.pdf) — a great short read
  introducing the Zipper
* [Weaving a web](http://dspace.library.uu.nl/bitstream/handle/1874/2532/2001-33.pdf) by Hinze and Jeuring —
  another interesting Zipper-like approach

#### Scala libraries

* [zipper](https://github.com/stanch/zipper) — my Zipper implementation
* [Monocle](https://github.com/julien-truffaut/Monocle) — an “optics” library
* [Quicklens](https://github.com/adamw/quicklens) — a simpler way to update nested case classes
* [FingerTree](https://github.com/Sciss/FingerTree) — an implementation of the Finger Tree data structure
