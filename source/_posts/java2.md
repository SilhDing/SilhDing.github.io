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

Apparently, the bug is because we use the value of variable `color` before it is initialized. So, always remember that ***it is possible to observe the value of a final instance field before its value has been assigned***.

In fact, this problem arises whenever a constructor calls a method that has been over-ridden in its subclass. A method invoked in this way always runs before the instance has been initialized, when its declared fields still have their default values. To avoid this problem, **never call overridable methods from constructors**.

### Sum Fun

What does this program print?

```Java
class Cache {
    static {
        initializeIfNecessary();
    }

    private static int sum;
    public static int getSum() {
        initializeIfNecessary();
        return sum;
    }

    private static boolean initialized = false;
    private static synchronized void initializeIfNecessary() {
        if (!initialized) {
            for (int i = 0; i < 100; i++)
                sum += i;
            initialized = true;
        }
    }
}

public class Client {
    public static void main(String[] args) {
        System.out.println(Cache.getSum());
    }
}
```

It seems that the author of this program apparently went to a lot of trouble to make sure that `sum` was initialized before use, and expected `Cache.sum` to be `4950`. However, the program tells you the sum is `9900`, fully twice the expected value.

Obviously, the problem is because of the class initialization. Let's go through the whole process.

Before it can invoke `Client.main`, the VM must initialize the class `Client`. The `Client.main` method invokes `Cache.getSum`. Before the `getSum` method can be executed, the VM must initialize the class `Cache`.

Do you still remember the order of static initializers? It follows the order they appear in the source. The `Cache` class has two static initializers: the `static` block at the top of the class and the initializing of the static field `initialized`. The first one will be executed first, adding 4950 to `sum` (default value of `sum` is `0`) and setting `initialized` to `true`.

The second initializer will com later. **The static initializer for the `initialized` field sets it back to `false`**, completing the class initialization of `Cache`. Thus, when calling `Cache.getSum()`, `sum` will be added with 4950 again.

From the program, it infers that the programmer did not think about the order in which the initialization of the `Cache` class would take place, and was unable to decide between eager and lazy initialization. The author used both eager (`initializeIfNecessary()` in static block) and lazy initialization (`initializeIfNecessary()` in `getSum()`). Remember: ***use either eager initialization or lazy initialization, never both***.

Instead, the code below is a better version for `Cache`.

```Java
class Cache {
    private static final int sum = computeSum();

    private static int computeSum() {
        int result = 0;
        for (int i = 0; i < 100l i++)
            result += i;
        return results;
    }

    public static int getSum() {
        return sum;
    }
}
```

A goof lesson (which we mentioned previously): if you want to compute a value for a static field, you'd better write a static function to do it.

### Creationism

What does this program print?

```Java
public class Creator {
    public static void main(String[] args) {
        for (int i = 0; i < 100; i++)
            Creature creature = new Creature();
        System.out.println(Creature.getNum());
    }
}

class Creature {
    private static long num = 0;
    public Creature() {
        num ++;
    }
    public static long getNum() {
        return num;
    }
}
```

Pretty trick, right? If you try to write this program on your favorite IDE, you will find that this program does not compile. My IDE will tell me "Declaration not allowed here" at line 5.

I would say this puzzle is the most impressive and surprising one. Please notice that, line 5 is *a local variable declaration statement*. The syntax of the language does not allow a local variable declaration statement as the statement repeated by a `for`, `while`, or `do` loop. ***A local variable declaration can appear only as a statement directly within a block***. (A block is a pair of curly braces and the statements and declaration contained within it.)

There are two ways to fix it:

```java
for (int i = 0; i < 100; i++) {
    Creature creature = new Creature();
}
```

```java
for (int i = 0; i < 100; i++)
    new Creature();
```

## Library Puzzlers

### The Dating Game

What does this program print?

```Java
public class DatingGame {
    public static void main(String[] args) {
        Calendar cal = Calendar.getInstance();
        cal.set(1999, 12, 31);
        System.out.println(cal.get(Calendar.YEAR) + " ");

        Date d = cal.getTime();
        System.out.println(d.getDay());
    }
}
```

It seems that the program should print `1999 31`. However, it prints `2000 1`. What is happening here?

Our program illustrates a few of the problems with the `Date` and `Calendar` API. The program's first bug is in the method invocation `cal.set(1999, 12, 31)`. **`Date` represents January as 0, and `Calendar` perpetuates this mistake**. Therefore, this method invocation sets the calendar to the thirty-first day of the thirteenth month of 1999. But the standard (Gregorian) calendar has only 12 months; surely this method invocation should cause an `IllegalArgumentException`? **It should, but it doesn't**. The `Calendar` class silently substitutes the first month of the next year, in this case, 2000. This explains the first number printed by  our program (`2000`).

Another bug is: **`Date.getDay` returns the day of the week represented by the `Date` instance, not the day of the month**. The returned value is 0-based, starting at Sunday.

A good thing is some of these confusing methods are deprecated already, and you should avoid using these methods. In addition, be careful when using `Calendar` or `Date` (or even other classes); always consult the API documentation.

### The Mod Squad

What does this program print?

```Java
public class Mod {
    public static void main(String[] args) {
        final int MODULUS = 3;
        int[] histogram = new int[MODULUS];

        int i = Integer.MIN_VALUE;
        do {
            histogram[Math.abs(i) % MODULUS] ++;
        } while (i ++ != Integer.MAX_VALUE);

        for (int j = 0; j < MODULUS; j++) {
            System.out.println(histogram[j] + " ");
        }
    }
}
```

We expect to see number from each entry of `histogram` is roughly equal. However, if you run this code. You will get:

```
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index -2 out of bounds for length 3
	at src.Mod.main(Mod.java:8)
```

So `Math.abs(i)` returns a negative value? Let's go to the documentation for `Math.abs`. The method is supposed to always return non-negative value, but in one case, it does not. The documentation says: if the argument is equal to the value of `Integer.MIN_VALUE`, the result is that same value, which is negative.

Please remember that `Math.abs()` is not guaranteed to return a non-negative value.

### A strange Saga of a Suspicious sort

What does this program print?

```Java
public class SuspiciousSort {
    public static void main(String[] args) {
        Random rnd = new Random();
        Integer[] arr = new Integer[100];

        for (int i = 0; i < arr.length; i++) {
            arr[i] = rnd.nextInt();
        }

        Comparator<Integer> cmp = new Comparator<>() {
            @Override
            public int compare(Integer i1, Integer i2) {
                return i2 - i1;
            }
        };

        Arrays.sort(arr, cmp);
        System.out.println(order(arr));
    }

    enum Order {ASCENDING, DESCENDING, CONSTANT, UNORDERED};

    static Order order(Integer[] a) {
        boolean ascending = false;
        boolean descending = false;

        for (int i = 1; i < a.length; i++) {
            ascending |= (a[i] > a[i-1]);
            descending |= (a[i] < a[i-1]);
        }

        if (ascending && !descending)
            return Order.ASCENDING;
        if (descending && ! ascending)
            return Order.DESCENDING;
        if (!ascending)
            return Order.CONSTANT;
        return Order.UNORDERED;
    }
}
```

In this program, we try to sort a random generated array with a self-defined comparator. Then, we use a static method `order` to determine the type of the order. From the code, the comparator is supposed to sort the array in descending order. However, if you run this code, you will find that the program will print `UNORDERED`. What happens here?

There is a problem in the comparator. Seemingly this idiom always makes sense: if you have two numbers and you want a value whose sign indicates their order, compute their difference. However, the problem with this idiom is that a fixed-width integer is not big enough to hold the difference of two arbitrary integers of the same width. When you subtract two `int` or `long` values, the result can overflow, in which case it will have the wrong sign. Try the program below:

```Java
public class OverFlow {
	public static void main(String[] args) {
		int x = -2000000000;
		int z = 2000000000;
		System.out.println(x - z);
	}
}
```

Thus, subtracting two values to compare them might not be a good choice.

<br>

------------ END OF THIS POST ------------

If you want to read more puzzlers, please go to [this](/2020/04/17/java3/) page!
