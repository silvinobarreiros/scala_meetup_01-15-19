# Functional Patterns In Practice
---

<img src="assets/eugene.jpg" style="height: 75%; width: 75%;"/>
---

## Who AMI

* Lead Engineer @ Stash

* Pug Enthusiast

* C -> Java -> C# -> Java -> Scala
---

## @pugsnaps on IG
<img src="assets/diesel.jpg" style="height: 35%; width: 35%;"/>
---

## What are we doing?

* Walk through 3ish of the 99 Scala problems

* Move us from imperative Java-y Scala to functional Scala-y Scala
---

## Scala-y Scala?

* avoiding `null`

* using `val` over `var`

* avoid mutating state
---

## Problem 1

Return the last element of a linked list

---?image=assets/linked_list.jpeg&size=auto 50%

---

### version 1
```scala
def last(list: List[Int]): Int = {
  var last = Int.MaxValue

  if (list.length == 1) {
    last = list(0)
  } else if (list.nonEmpty) {
    last = list(list.length - 1)
  }

  last
}
```
@[2](`var` 👎🏼)
@[5, 7](mutating state)
@[2](Max Int as a base case??)

---

### version 2
```scala
def last(list: List[Int]): Int = {
  if (list.length == 1) {
    list(0)
  } else if (list.nonEmpty) {
    list(list.length - 1)
  } else {
    Int.MinValue
  }
}
```
@[2-8]("everything" is an expression 🙌🏼)
@[7](Min Int any better?)

---

## Option Break

```scala
trait Option[+A] {
  def get: A
}

case class Some[A](item: A) extends Option[A]
case object None extends Option[A]
```
@[1-3](Option!)
@[5](Value exists)
@[6](No value)

---

### version 3
```scala
def last[A](list: List[A]): Option[A] = {
  if (list.length == 1) {
    Some(list(0))
  } else if (list.nonEmpty) {
    Some(list(list.length - 1))
  } else {
    None
  }
}
```
@[1](return an `Option`)
@[7](base case `None` 🎉)
@[1-9](`None` has a semantic meaning)

---

## Pattern Matching Break

```scala
import scala.util.Random

val x: Int = Random.nextInt(10)

x match {
  case 0 => "zero"
  case 1 => "one"
  case 2 => "two"
  case _ => "many"
}
```

@[5](match on random number)
@[6](x == 0)
@[7-8](x is 1 or 2)
@[9](match any other int)
---

## Pattern Matching Break

```scala
abstract class Device

case class Phone(model: String) extends Device {
  def screenOff = "Turning screen off"
}

case class Computer(model: String) extends Device {
  def screenSaverOn = "Turning screen saver on..."
}

def goIdle(device: Device) = device match {
  case p: Phone => p.screenOff
  case c: Computer => c.screenSaverOn
}
```
@[1-10](3 types)
@[11-14](device match)
@[12](we have a Phone!)
@[13](and a Computer!)
---

### version 4
```scala
def last[A](list: List[A]): Option[A] = list match {
  case Nil => None
  case head :: Nil => Some(head)
  case head :: tail => last(tail)
}
```

@[2](empty list case)
@[3](we have 1 element)
@[4](we have more than one element)
@[4](List(1, 2, 3) --> head = 1, tail = List(2, 3))

---

### lawls...
```scala
def last[A](list: List[A]): A = list.last
```
---

## Problem 15

Duplicate the elements of a list a given number of times

```
duplicateN(2, List("a", "b", "c"))
...
List("a", "a", "b", "b", "c", "c")
```
---

### version 1
```scala
def duplicateN[A](n: Int, list: List[A]): List[A] = {
  import scala.collection.mutable.ListBuffer
  
  val buffer = new ListBuffer[A]()
  var i = 0

  for (i <- 0 until list.length) {
    var j = 0

    for (j <- 0 until n) {
      buffer += list(i)
    }
  }

  buffer.toList
}
```

@[5, 8](TWO `var`s 😱)
@[4, 9, 13](mutating state)

---

### version 2
```scala
def duplicateN[A](n: Int, list: List[A]): List[A] = {
  import scala.collection.mutable.ListBuffer

  val buffer = new ListBuffer[A]()
  var i = 0

  for (i <- 0 until list.length) {
    val dups = List.fill(n)(list(i))
    
    dups.foreach(j => buffer += j)
  }

  buffer.toList
}
```

@[8](removed one loop)
@[10](still mutating state 🤭)

---

## FoldLeft break

```scala
def foldLeft[B](z: B)(f: (B, A) => B): B
```
---

### Quick example

```scala
val list = List(1, 2, 3)

val result = list.foldLeft(0) { (total, num) => 
  b + a 
}
```

1. total = 0, num = 1
2. total = 1 (0 + 1), num = 2
3. total = 3 (2 + 1), num = 3
4. result = 6
---

### version 3
```scala
def duplicateN[A](n: Int, list: List[A]): List[A] = {
  
  list.foldLeft(List.empty[A]) { (acc, next) =>
    acc ++ List.fill(n)(next)
  }
}
```

@[4](acc = List(), next = "a")
@[4](acc = List("a", "a"), next = "b")
@[4](acc = List("a", "a", "b", "b"), next = "c")
@[4](result = List("a", "a", "b", "b", "c", "c"))

@[4](new copy of the list is returned every iteration)
---

### version 4
```scala
def duplicateN[A](n: Int, list: List[A]): List[A] = {
  val res = list.map { item =>
    List.fill(n)(item)
  }

  res.flatten
}
```

@[2](List( List("a", "a"), List("b", "b"), List("c", "c") ))
@[6](turn the list of lists into a single list 🔥)

---

### version 5
```scala
def duplicateN[A](n: Int, list: List[A]): List[A] = 
  list.flatMap(List.fill(n)(_))
```

map + flatten == flatMap
---

## Problem 61

Count the leaf nodes of a binary tree

---?image=assets/tree.png&size=auto 75%
---

## Case Classes

```scala
case class Node[+T](value: T, left: Node[T], right: Node[T])
```
---

## Java Equivalent
```java
public class Node<T> {

  private final T value;
  private final Node<T> left;
  private final Node<T> right;

  public Node(T value, Node<T> left, Node<T> right) {
    this.value = value;
    this.left = left;
    this.right = right;
  }

  public T getValue() {
    return this.value;
  }

  public Node<T> getLeft() {
    return this.left;
  }

  public Node<T> getRight() {
    return this.right;
  }

  @Override
  public boolean equals(Object o) {
    // dis == dat??
  }

  @Override
  public int hashcode() {
    // super sick hashcodez
  }
}
```

@[3-5](instance variables)
@[7-11](constructor)
@[13-23](getters)
@[25-28](instance level equals)
@[30-33](oh and hashcode)
---

```scala
case class Node[+T](value: T, left: Node[T], right: Node[T])
```
---

### version 1
```scala
def leafCounter[T](node: Node[T]): Int = {
  import scala.collection.mutable

  val queue = new mutable.Queue[Node[T]]
  var temp = node
  var count = 0

  while(temp != null) {
    if (temp.left != null) {
      queue += temp.left
    }

    if (temp.right != null) {
      queue += temp.right
    }
```
---

### version 1
```scala
    if (temp.right == null && temp.left == null) {
      count = count + 1
    }

    temp = if (queue.nonEmpty) queue.dequeue() else null
  }

  count
}
```
---

### version 1
```scala
def leafCounter[T](node: Node[T]): Int = {
  import scala.collection.mutable

  val queue = new mutable.Queue[Node[T]]
  var temp = node
  var count = 0

  while(temp != null) {
    if (temp.left != null) {
      queue += temp.left
    }

    if (temp.right != null) {
      queue += temp.right
    }

    if (temp.right == null && temp.left == null) {
      count = count + 1
    }

    temp = if (queue.nonEmpty) queue.dequeue() else null
  }

  count
}
```

@[5-6](`var`s...)
@[8, 9, 13, 17, 21](THERE.ARE.SO.MANY.NULLS)
@[4, 10, 14, 18](mutating state all over the place)
---

### version 2
```scala
def leafCounter[T](node: Node[T]): Int = {
  if (node == null) {
    0
  } else if (node.left == null && node.right == null) {
    1
  } else {
    leafCounter(node.left) + leafCounter(node.right)
  }
}
```

@[1-9](No more mutable queue)
@[1-9](Less code 🙌🏼)
---

### version 3
```scala
def leafCounter[T](node: Node[T]): Int = {
  node match {
    case null => 0
    case Node(_, null, null) => 1
    case Node(_, left, right) => 
      leafCounter(left) + leafCounter(right)
  }
}
```
@[3](node is null)
@[4](leaf node)
@[5-6](left and right exist)
@[3-4](still hanging out with those `null`s 😶)

---

### version 4

```scala
trait Tree[+T]

case class Node[+T](
  value: T, 
  left: Tree[T], 
  right: Tree[T]) extends Tree[T]

case object End extends Tree[Nothing]
```

@[1](trait == interface)
@[3-6](same case class but now extends `Tree`)
@[8](singleton)
---

### version 4

```scala
def leafCount[T](tree: Tree[T]): Int = {
  tree match {
    case Node(_, End, End) => 1
    case Node(_, left, right) => 
      leafCount(left) + leafCount(right)
    case _ => 0
  }
}
```

@[3](both left and right ARE `End`)
@[4-5](both left and right are not `End`)
@[6](everything else)
@[1-8]()
---

## Questions

😬
