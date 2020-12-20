---
title: "A Java programmer? Try these puzzles! (Part 3/3)"
date: 2020-04-17 23:20:02
tags:
    - "Java"
categories:
    - "technique"
    - "programming"
---

Let's continue our puzzlers!

## Classier Puzzlers

Puzzlers in this chapter will concern inheritance, overriding, and other forms of name reuse.

### A Private Matter

In this program, a subclass filed has the same name as a superclass field. What does the program print?

```Java
class Base {
    public String className = "Base";
}

class Derived extends Base {
    private String className = "Derived";
}

public class PrivateMatter {
    public static void main(String[] args) {
        System.out.println(new Derived().className);
    }
}
```

There are two possible behavior you may expect from this program:

1. it should print `Base` as overriding may not take effect;
2. it does not compile, because the variable className in Derived has more restrictive access than it does in `Base`.

If you run this program, you will find it does not compile, but the program is with `PrivateMatter`. Had `className` been an instance method instead of an instance filed, `Derived.className()` would have *overridden* `Base.className()`, and the program would have been illegal. The access modifier of an overriding method must provide at least as much access as that of the overridden method. Because `classname` is a field, `Derived.className` *hides* `Base.className` rather than overriding it. It is legal, though inadvisable.

In fact, hiding always cause confusion. A class that hides a field with one whose accessibility is more restrictive than that of the hidden field, as in out original program, violates the principle of *subsumption*, also known as *Liskov Substitution Principle*. This principle, generally speaking, says that everything you can do with a base class, you can also do with a derived class. This is an integral part of the natural mental model of object-oriented programming. The program will become very difficult to understand if it is violated. This principle is also violated when hiding one filed with another and:

- if the two field are of different types;
- if one field is static and the other isn't;
- if one field is final and the other isn't;
- if one filed is constant and the other isn't;
- of both are constant and have different values.

### All Strung Out

Try to run the program below with the Java command or your favorite IDE. **Don't just read it and guess what is the output**.

```Java
public class StrungOut {
    public static void main(String[] args) {
        String s = new String("Hello world!");
        System.out.println(s);
    }
}

class String {
    private final java.lang.String s;

    public String(java.lang.String s) {
        this.s = s;
    }

    public java.lang.String toString() {
        return s;
    }
}
```

This program looks simple. It acts like a wrapper for `String`. However, if you try to run this code from an IDE (such as IntelliJ), it will not allow you to run it; if you run with java command, if will give you an exception:

    Exception in thread "main" java.lang.NoSuchMethodError: main

It seems strange, because we indeed have a main method in class `StrungOut`. Why can't the VM find it?

Although `StrungOut` has a method named "main", it has the wrong signature. A `main` method must accept a single argument that is an array of strings. What the VM is struggling to tell us is that `StrungOut.main` accepts an arrays of *our* `String` class, which has nothing whatsoever to do with `java.lang.String`.

Remember: ***avoid reusing the names of platform classes, and never reuse class names from java.lang***.

### Package Deal

This program involves the interaction of two classes in different packages. What does the program print?

```Java
package hack;
import click.CodeTalk;

public class TypeIt {
    private static class ClickIt extends CodeTalk {
        void printMessage() {
            System.out.println("Hack");
        }
    }

    public static void main(String[] args) {
        ClickIt clickit = new ClickIt();
        clickit.doIt();
    }
}
```

```Java
package click;
public class CodeTalk {
    public void doIt() {
        printMessage();
    }

    void printMessage() {
        System.out.println("Click");
    }
}
```

If you run this program, you will find the output is `Click`. In fact, ***a package-private method cannot be directly overridden by a method in a different package***. The two `printMessage` methods in this program are unrelated; they merely have the same name. The purpose of this rule for Java is to enable packages to support encapsulation of abstractions larger than a single class.

If you want the `printMessage` method in `hack.TypeIt.ClickIt` to override the method in `click.CodeTalk`, you must add the `protected` or `public` modifier to the method declaration in `click.CodeTalk`. To make the program compile, you must also add a modifier to the overriding declaration in `hack.TypeIt.ClickIt`. The modifier must be no more restrictive than the one you placed in the declaration for `printMessage` in `hack.TypeIt.ClickIt`.

### Import Duty

What does the program below print?

```Java
import static java.util.Arrays.toString;

public class ImportDuty {
    public static void main(String[] args) {
        printArgs(1,2,3,4,5);
    }

    static void printArgs(Object... args) {
        System.out.println(toString(args));
    }
}
```

However, this program does not compile. What is the problem? Is it because the compiler cannot find the function `toString`?

If you write this program in an IDE, probably the IDE will tell you what is the result: "Non-static method `toString` cannot be referenced from a static context". The function `toString` we used in function `printArgs` is in fact the member method of class `ImportDuty` (inherit this method from class `Object`).

***Members that re naturally in scope take precedence over static imports***. Remember: static import facility cannot be used effectively on a method if its name is already in scope.