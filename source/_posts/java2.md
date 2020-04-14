---
title: "A Java programmer? Try these puzzles! (Part 2/3)"
date: 2020-04-13 21:59:26
tags:
    - "Java"
categories:
    - "technique"
    - "programming"
---

Let's continue our puzzlers!

## Classy Puzzlers

### The Case of the Confusing Constructor

What does the program below print?

```Java
public class Confusing {
    private Confusing(Object o) {
        System.out.println("Object");
    }
    private Confusing(double[] dArray) {
        System.out.println("double array");
    }

    public static void main(String[] args) {
        new Confusing(null);
    }
}
```

Apparently, this is a question on overloading. Which method is called by `main` method? The answer is the second one: `Confusing(double[])`.

There are two phases in Java's overload resolution process:
1. selects all the methods or constructors that are accessible and applicable;
2. selects the *most specific* of the methods or constructors selected in the first phase.

In our program, both constructors are applicable and accessible, but `Confusing(Object)` is less specific as accepts any parameter passed to `Confusing(Object)`. The key to understanding this puzzle is that the test for which method or constructors is most specific does not use the *actual parameters*: the parameters appearing in the invocation (e.g., null in this case).

This puzzle should indicate some advices for you when you are writing some APIs: ensure your clients aren't forced to go to these extreme fashions of selecting among overloadings. Ideally, you should ***avoid*** overloading: use different names for different methods, which, however, it is not possible sometimes. For example, constructors don't have names, but you could make constructors private and providing public static libraries. If constructors have too many parameters, consider builder pattern.

If you really have to do overload, make sure that all overloading accept mutually incompatible parameter types, ***so that no two are applicable at the same time***.

### Well, Dog My Cats!

What does this program print?

```Java
class Counter {
    private static int count;
    public static void increment() { count ++; }
    public static int getCount() { return count; }
}

class Dog extends Counter {
    public Dog() { }
    public void woof() { increment(); }
}

class Cat extends Counter {
    public Cat() { }
    public void meow() { increment(); }
}

public class Ruckus {
    public static void main(String[] args) {
        Dog[] dogs = { new Dog(), new Dog() };
        for (int i = 0; i < dogs.length; i++) {
            dogs[i].woof();
        }
        Cat[] cats = {new Cat(), new Cat(), new Cat()};
        for (int i = 0; i < cats.length; i++) {
            cats[i].meow();
        }
        System.out.print(Dog.getCount() + " woofs ");
        System.out.println(Cat.getCount() + " meows");
    }
}
```

A naive guess is `2 woofs 3 meows`. However, it turns out to be `5 woofs 5 meows`. It is not easy to get the reason: a single copy of each static field is shared among its declaring class and all subclasses. So `Dog` and `Cat` use the same `count` field.

To avoid this problem, we must correct a fundamental design error. When designing one class to build on the behavior of another, you have two options: *inheritance* (one class extends the other) and *composition* (one class contains an instance of the other). Apparently, composition is preferable in this case.

```Java
class Dog {
    private static Counter counter = new Counter();
    public Dog() {}
    public static int woofCount() { return counter.getCount();}
    public void woof() {counter.increment();}
}
```

If you are familiar with design patterns and principles in Java, you will aware that composition uses delegation to assign tasks and reduce coupling. You can refer to *Effective Java* for more information.

### All I Get Is Static

What does the program below print?

```Java
class Dog {
    public static void bark() {
        System.out.println("Woof ");
    }
}

class Basenji extends Dog {
    public static void bark() { }
}

public class Bark {
    public static void main(String[] args) {
        Dog woofer = new Dog();
        Dog nipper = new Basenji();
        woofer.bark();
        nipper.bark();
    }
}
```

Both `woofer` and `nipper` are declared with `Dog`, but `nipper` is initializer with `Basenji` constructor. The method `bark` is `Basenji` is defined to do nothing. So `nipper.bark()` will print nothing. Is this true?

No. You will see two `Woof ` on the screen.

The problem is that `bark` is a static method, and **there is no dynamic dispatch on static methods**. When a program calls a static method, the method to be invoked is selected at compile time, based on the compile-time type of the qualifier, which is the name we give to the part of the method invocation expression to the left of the dot. In this program, both `woofer` and `nipper` are declared to be of type `Dog` and they have to same compile-time type, the compiler causes the same method to be invoked: `Dog.bark`. It does not matter that the runtime type of `nipper` is `Basenji`; only compile-time type is considered.

When you  invoke a static method, you typically qualify it with a class rather an expression.

    Dog.bark();
    Basenji.bark();

In fact, this is what Java recommends to do. Remember: ***static methods cannot be overridden***.

### Larger Than Life

What does the program below print?

```Java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private final int beltSize;
    private static final int CURRENT_YEAR =
        Calendar.getInstance().get(Calendar.YEAR);

    public Elvis() {
        beltSize = CURRENT_YEAR - 1930;
    }

    public int getBeltSize() {
        return this.beltSize;
    }

    public static void main(String[] args) {
        System.out.println(
            "Elvis wears a size " + INSTANCE.getBeltSize() + " belt.");
    }
}
```

The program will return you `Elvis wears a size -1930 belt.`. It seems that `INSTANCE.getBeltSize()` will return `0`.

If you still have some memory of the puzzler "The Reluctant Constructor", you might have some idea of what is really happening. However, `INSTANCE` here is a static variable and it may not cause `StackOverFlowError`. But the reason is similar: we have to dive deep into order of *class* initialization.

Initialization of `Elvis` is triggered by the VM's call to its `main` method. First, static field are set to their default values. The field `INSTANCE` is set to `null`, and `CURRENT_YEAR` is set to 0. Next, static field initializers are executed in order of appearance. The first static field is `INSTANCE`. Its value is computed by invoking the `Elvis()` constructor.

The constructor initializes `beltSize` to an expression involving the static field `CURRENT_YEAR`. Normally, reading a static field is one of the things that causes a class to be initialized, but we are already initializing the class `Elvis`. Recursive initialization attempts are simply ignored. Consequently, the value of `CURRENT_VALUE` still has its default value of 0. That is why Elvis's belt size if `-1930`. Though `CURRENT_YEAR` will be initialized to `2020` (current year), it is too late for the now correct value of this field to affect the computation of `Elvis.INSTANCE.beltSize`.

This puzzle shows that ***it is possible to observer a final static field before it is initialized.*** You should always be careful of the initialization cycles.

### What's the Point?

What does this program print?

```Java
class Point {
    private final int x, y;
    private final String name;

    Point(int x, int y) {
        this.x = x;
        this.y = y;
        name = makeName();
    }

    protected String makeName() {
        return "[" + x + "," + y + "]";
    }

    public final String toString() {
        return name;
    }
}

public class ColorPoint extends Point {
    private final String color;

    ColorPoint(int x, int y, String color) {
        super(x, y);
        this.color = color;
    }

    protected String makeName() {
        return super.makeName() + ":" + color;
    }

    public static void main(String[] args) {
        System.out.println(new ColorPoint(4, 2, "red"));
    }
}
```

It is not hard to spot the bug in the program, but I believe many programmers might not notice this bug. The result returned by this program is `[4,2]:null`.

If you are not aware of the issue, let's mark the order of instance initialization.

```Java
class Point {
    private final int x, y;
    private final String name;

    Point(int x, int y) {
        this.x = x;
        this.y = y;
        name = makeName();  // 3. Invoke subclass method
    }

    protected String makeName() {
        return "[" + x + "," + y + "]";
    }

    public final String toString() {
        return name;
    }
}

public class ColorPoint extends Point {
    private final String color;

    ColorPoint(int x, int y, String color) {
        super(x, y);  // 2. Chain to Point constructor
        this.color = color;  // 5. Initialization blank final-Too late
    }

    protected String makeName() {
        //  4. Executes before subclass constructor body!
        return super.makeName() + ":" + color;
    }

    public static void main(String[] args) {
        //  1. Invoke subclass constructor
        System.out.println(new ColorPoint(4, 2, "red"));
    }
}
```

Apparently, the bug is because we use the value of variable `color` before it is initialized. So, always remember that ***it is possible to observe the value fo a final instance field before its value has been assigned***.
