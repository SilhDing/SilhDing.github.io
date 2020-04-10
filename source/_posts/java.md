---
title: "A Java programmer? Try these puzzles!"
date: 2020-04-05 16:08:33
tags:
    - "Java"
categories:
    - "technique"
    - "programming"
---

![book](book.jpg)

<br>

If you are a Java programs, ***I highly recommend reading this book: Java Puzzler***, by *Joshua Bloch* and *Neal Gafter*. This book will introduce many common traps, pitfalls and corner cases that you are highly likely to meet when programming with Java.

This post is intended to summarize the most interesting and critical pitfalls recorded in this book. Trust me, you will be highly surprised with some of them.
<br>

![gif](pitfall.gif)

<br>
This post will keep updating. Welcome to revisit!

## Expressive Puzzlers

### The Joy of Hex

What does the program print?

```java
public class JoyOfHex {
    public static void main(String[] args) {
        System.out.println(Long.toHexString(0x100000000L + 0xcafebabe));
    }
}
```

The most intuitive answer would be `1cafebabe`, right? However, if you run this program, it is in fact `cafebabe`.

A very subtle feature for decimal literal is: ***decimal literals are all positive***. This feature is not shared by hexadecimal or octal literals. To write a negative decimal literal, we always use a minus sign `-` in combination with a decimal literal. Thus, negative decimal constants are clearly identifiable by the presence of the minus sign. ***Hex and octal literals are negative if their high-order bit is `1`***.

In this program, the number `0xcafebabe` is an `int` constant with its high-order bit set, so it is negative. The addition performed by the program is a mixed-type computation: the left operand is of type `long`. Java will promote the `int` value to a `long` value with a *widening primitive conversion*.

Final computation will be: `0xffffffffcafebabeL + 0x0000000100000000L = 0x00000000cafebabeL`

### Swap Meat

What does this program print?

```java
public class CleverSwap {
    public static void main(String[] args) {
        int x = 1984;  //(0x7c0)
        int y = 2001;  //(0x7d1)
        x ^= y ^= x ^= y;
        System.out.println("x = " + x + "; y = " + y);
    }
}
```

As referred by its name, this program is supposed to swap the values of the variables `x` and `y`. However, you will find it fails, printing `x = 0; y = 1984`.

The most natural way to swap two variables is to use a temporary variable:

```java
int tmp = x;
x = y;
y = tmp;
```

It was discovered that one could avoid the temporary variable:

```java
x = x ^ y;
y = y ^ x;
x = y ^ x;
// promise me: don't do this in your program :)
```

In fact, it works. However, it is not recommended, as it is far less clear than its naive counterpart and far slower. Now, you might combine the three exclusive OR operations and make it into a single statement, like in the code.

However, it is guaranteed not to work in Java, though might work in other languages (C, C++, ...).

Operands of operators are evaluated from left to right. To evaluate the expression `x ^= expr`, the value of `x` is sampled **before** `expr` is evaluated, and the exclusive OR of these two values is assigned to the variable `x`.

The code below describes what in fact happens in the original program:

```java
int tmp1 = x;  // first appearance of x in the expression
int tmp2 = y;  // first appearance of y
int tmp3 = x ^ y;  // compute x ^ y
x = tmp3;  // last assignment: store x ^ y in x
y = tmp2 ^ tmp3;  // second assignment, store original x value in y
x = tmp1 ^ y;  // first assignment: store 0 in x
```

If you really want to make it work, you shoudl write it like:

```java
y = (x ^= (y ^= x)) ^ y;  // NOT recommended!
```

A lesson: do not assign to the same variable more than once in a single expression.

### Dos Equis

Now look at an example of conditional operator. What does this program print?

```Java
public class DosEquis {
    public static void main(String[] args) {
        char x = 'X';
        int i = 0;
        System.out.print(true ? x : 0);
        System.out.print(false ? i : x);
    }
}
```

Okay, this program seems trivial, and both `print` lines will print out `x`. However, if you run the program, it turns out to be `X88`.

I would admit this is the most surprising behavior of Java I've ever seen.

The answer lies in a dark corner of the specification for the conditional operator. ***Mixed-type computation can be confusing. Nowhere is this more apparent than in conditional expressions.***

So, this problem is due to that these two expressions have different result types (one is `char` is one is `int`). There are three key points in determining the result type of a conditional expression:

1. If the second and third operands have the same type, that is the type of the conditional expression;
2. If one of the operands is of type *T* and *T* is `byte`, `short`, or `char` and the other operand is a constant expression of type `int` whose value is represented in type *T*, the type of the conditional expression is *T*.
3. Otherwise, binary numeric promotion is applied to the operands types, and the type of the conditional expression is the promoted type of the second and third operands.

Remember: use the same type for the second and third operands in conditional expressions.

## Puzzlers with characters

### Animal Farm

What does this program print?
```Java
public class AnimalFarm {
    public static void main(String[] args) {
        final String pig = "length: 10";
        final String dog = "length: " + pig.length();
        System.out.println(pig == dog);
    }
}
```

Both strings will be `length: 10` in this case. Though the `==` operator does not test whether two objects are equal (it tests whether two object reference are identical), compile-time constants of type `String` are **interned**. So this program should print `True`, right?

Unfortunately, no. ***This is because the second string is not initialized with constant expressions***, and they will not point to the a same object.

### Escape Rout

The following programs uses two Unicode escapes. What does this program print?

```Java
public class EscapeRout {
    public static void main(String[] args) {
        // \u0022 is the Unicode escape for double quote (")
        System.out.println("a\u0022.length() + \u0022b".length());
    }
}
```

You probably guess the result is 16 or 26. However, it's 2.
