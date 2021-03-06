---
layout: post
title: Methods invocation inside of a Java Virtual Machine
tags: [JVM]
---

[Java Virtual Machine](https://en.wikipedia.org/wiki/Java_virtual_machine) (JVM) is an abstract virtual machine that executes [Java bytecode](https://en.wikipedia.org/wiki/Java_bytecode). JVM has a number of [implementations](https://en.wikipedia.org/wiki/List_of_Java_virtual_machines), the most popular being [HotSpot JVM](https://en.wikipedia.org/wiki/HotSpot), which will be used as an example of the implementation throughout the article. Initially, JVM was designed at [Sun Microsystems](https://en.wikipedia.org/wiki/Sun_Microsystems) to execute [Java](https://en.wikipedia.org/wiki/Java_(software_platform)) programs. Despite the fact that Java language and JVMs are influencing each other’s designs, engineers that work on JVMs are trying to make them as generic as possible. For this reason, JVM is a great execution engine for languages other than Java, such as  [Scala](https://www.scala-lang.org/) and [Kotlin](https://kotlinlang.org/), which are statically typed, and [Groovy](http://groovy-lang.org/) and [Ruby](http://ruby-lang.org/), which are dynamically typed languages. In this article, I will take a look at how JVM handles method invocation and its support for static and dynamic programming languages.

## Methods constraints on JVM

Generally speaking, all programs that are written in the wild are a bunch of methods structured in some manner to solve a particular problem. Java bytecode is an interface between a programming language and JVM as an execution machinery. Any language that runs on JVM should have a compiler or a runtime that produces a valid bytecode that JVM understands and executes. Methods are the main artifacts of a programming language. Let's take a look at a Java method with a few arguments of different types, and see what bytecode `javac` generates to execute it on JVM:

```java
private static void someMethod(int i, String s, long l, Object o) {
}
```

We’ll compile it with `javac` and read with `javap` program to see the bytecode representation of the method:

```bc
//... some bytecode is omitted
private static void someMethod(int, java.lang.String, long, java.lang.Object);
  descriptor: (ILjava/lang/String;JLjava/lang/Object;)V
  flags: (0x000a) ACC_PRIVATE, ACC_STATIC
  Code:
    stack=0, locals=5, args_size=4
        0: return
//... some bytecode is omitted
```

Any method has to have a descriptor, access flags, and code. We will take a look at methods structure in the next sections of this article. What I want to focus on right now is that JVM requires that all methods have a descriptor with classes/types - thus static languages are in a better situation than dynamic ones. Static languages need to have a smart compiler that will produce valid bytecode, whereas, dynamic languages perform a lot of tricks to satisfy JVM requirements of a method bytecode. Let's take a look at how JVM deals with method invocation and its execution. Then, we’ll take a closer look at a dynamic languages example and what developers can do to overcome constraints that JVM enforces.

## JVM Method Frame

Let's get started with methods structure at runtime. Every time JVM invokes a method it creates a frame for the method on a current thread stack. JVM method frame contains local variables array, operand stack and reference to a current class constant pool. A local variables array is used to store method's local values and method's argument values. JVM allocates memory for local variables array and puts all method arguments at the beginning of the array. The size of the array is known during execution time, and is precomputed before the runtime phase. You should have seen this in `Code:` section with `locals=5` in the previous section example. The local variable array contains values of local variables for primitives and references to objects on the heap. JVM uses the operand stack to execute bytecode instruction. JVM either executes bytecode instruction that pushes an operand on the stack or pops needed operands number for an instruction and executes the instruction. The size of the operand stack is also precomputed. It can be seen in `Code:` section as `stack=0` in the code snippet above. Thus, JVM has the exact information on how much memory it needs to allocate on a thread stack for a particular method. A class constant pool is a kind of storage of different data, such as names of the other classes and methods, variable types and so on. It serves for linking other parts of the runtime image with a current method. Let's have a look how JVM executes bytecodes operations with a simple example of the sum of two numbers. Here is the Java code:

```java
int a = 1;
int b = 2;
int c = a + b;
```

And the following is the bytecode representation:

```bytecode
0: iconst_1
1: istore_1
2: iconst_2
3: istore_2
4: iload_1
5: iload_2
6: iadd
7: istore_3
```

When JVM encounters `iconst_1` instruction it pushes `1` on the operand stack. `i` in `iconst_1` means `integer` type, `const` - constant, `1` - a value of the constant. (JVM bytecode instruction set contains such instructions for constants up to value `5`). Then it reads the next instruction which is `istore_1`. This tells JVM to pop a value from the stack and store it into the array of local variables under index `1`. Then JVM repeats the same operation with the constant `2`. Instructions `iload_1` and `iload_2` tell JVM to push values from the local variables array at indexes `1` and `2` the operand stack; and `iadd` pops both variables from the operand stack, adds them and pushes the result on the operand stack. `istore_3` simply pops the result of addition from operand stack and stores it to the local variables array under the third index.

## JVM methods invocations

The JVM instruction set has 5 different bytecode instructions for a method invocation. In this part of the article, I briefly look at four of them: `invokestatic`, `invokeinterface`, `invokevirtual` and `invokespecial` - the fifth, `invokedynamic`, deserves a dedicated part of the article. Let's have a look at the first four instructions.

#### `invokestatic`

`invokestatic` instruction means the method that will be invoked is a method of a class not an instance method. Also, they are called `static` because you need to use `static` keyword in Java to say that a method is a class method, not an instance method.

#### `invokespecial`

`invokespecial` instruction indicates an invocation of `private` instance methods, `superclass` methods or `constructor`s.

#### `invokevirtual`

`invokevirtual` instruction is generated for methods with `public`, `package private or default` and `protected` access modifiers. Those methods are virtual and could be overridden by subclasses.

#### `invokeinterface`

`invokeinterface` instruction tells JVM that it should invoke a class implementation of an interface method.

## JVM method invocation instructions closer look

It seems a little odd that JVM specifies 4 different instructions for the only one purpose - just a method invocation. We need to consider what happens before JVM starts a method invocation to understand why it has so many variations. Before invoking a method, JVM performs two actions: method resolution and method lookup. Let's have a look at these actions separately. Every method invocation instruction has an index to an entry in a class constant pool. The entry consists of a method name and a method descriptor. You can find this data in decompiled java code. Let's examine the code below:

```java
public class Main {
  public static void main(String[] args) {
    someMethod(1, "2", 3L, new Object());
  }

  private static void someMethod(int i, String s, long l, Object o) {
  }
}
```

The code snippet can be compiled with a command

```sh
javac Main.java
```

and then decompiled with a command

```sh
javap -c -p -v Main.class
```

After that we can see the following class constant pool and bytecode instructions of `main` method.

```
//...
Constant pool:
   #1 = Methodref          #5.#18         // java/lang/Object."<init>":()V
   #2 = String             #19            // 2
   #3 = Long               3l
   #5 = Class              #20            // java/lang/Object
   #6 = Methodref          #7.#21         // Main.someMethod:(ILjava/lang/String;JLjava/lang/Object;)V
   #7 = Class              #22            // Main
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               main
  #13 = Utf8               ([Ljava/lang/String;)V
  #14 = Utf8               someMethod
  #15 = Utf8               (ILjava/lang/String;JLjava/lang/Object;)V
  #16 = Utf8               SourceFile
  #17 = Utf8               Main.java
  #18 = NameAndType        #8:#9          // "<init>":()V
  #19 = Utf8               2
  #20 = Utf8               java/lang/Object
  #21 = NameAndType        #14:#15        // someMethod:(ILjava/lang/String;JLjava/lang/Object;)V
  #22 = Utf8               Main
//...
 0: iconst_1
 1: ldc           #2                  // String 2
 3: ldc2_w        #3                  // long 3l
 6: new           #5                  // class java/lang/Object
 9: dup
10: invokespecial #1                  // Method java/lang/Object."<init>":()V
13: invokestatic  #6                  // Method someMethod:(ILjava/lang/String;JLjava/lang/Object;)V
16: return
```

> `javap` will generate a method name and its descriptor in a comment next to the method invocation

The `13: invokestatic  #6` points to the sixth entry in the class constant pool that contains the other two indexes `#6 = Methodref #7.#21`. `#7` is resolved into a string value of class name `Main` and `#21` into the string representation of method name `someMethod` and string value of the method descriptor `(ILjava/lang/String;JLjava/lang/Object;)V`. Parameter types are enclosed by parentheses and the method return type is after them. In this particular example `I` symbol stands for `int` type, `Ljava/lang/String` - a reference to an instance of `java.lang.String`, `J` - `long` and `Ljava/lang/Object` - a reference to a `java.lang.Object` instance.

Having this data, JVM now performs a method resolution which is a process of finding if the class has the method with the method name and its attributes that have been specified in the method descriptor. At this point, JVM proceeds with a method lookup, which is an operation of finding the method's index in the class methods table. Knowing that, let's dig into how HotSpot JVM takes advantage of these facts.

#### `invokestatic`

When HotSpot JVM encounters this instruction for the first time it performs a method resolution and lookup. However, as `static` methods can't be overridden, HotSpot wires method invocation inside a caller, and when JVM executes current method for next time it does not perform either the method resolution or the lookup - rather, it immediately invokes the method.

#### `invokespecial`

When HotSpot JVM comes across `invokespecial` instruction it performs the same procedure as for `invokestatic`, and, in addition, it checks that a class instance is not null every time the method is invoked.

#### `invokevirtual`

All Java developers know that instance methods can be overridden and the method signature in a subclass has to match the method signature in a superclass. HotSpot JVM saves the subclass method at the same index in a method table as in the superclass. As such, HotSpot JVM after a method resolution can save the method index and use it for method lookup the next time the method is invoked.

#### `invokeinterface`

Given that Java class can implement many interfaces, JVM manages a separated method table per class with interface implementation methods. As a result, when HotSpot JVM encounters an invokeinterface instruction, it completes both method resolution and method lookup every time a method is invoked.

## Invokedynamic

`invokedynamic` is a fairly new bytecode instruction added in JVM 1.7. Before diving deeper, let's clarify the reason why `invokedynamic` was added to the bytecode instruction set.

#### A dynamic language problem

JVM is a great execution runtime, however, dynamic languages have a problem when it is implemented. The problem is that JVM requires a strict signature of a method before invocation, an explicit receiver type (class or its instance), parameters types and a return type. Let's take a look at an example with a Ruby code

```ruby
def add(a, b)
  a + b
end
```

In the example above, `a` and `b` are method's parameters and `+` is a method on object `a`. Because of its dynamic nature, Ruby checks `a` and `b` types during runtime and is unable to provide required information to JVM ahead of time. Thus JRuby, which is a Ruby implementation on JVM, generates bytecode methods from Ruby code and manages its own methods tables with references to method bodies. One of the ways to get a reference to a method is JVM reflection API.

#### Reflection

JVM provides a way to invoke a method indirectly - to do this you need to know a `Class`, a method name and list of parameter types. Here is a trivial example of this procedure:

```java
Method substring = String.class.getDeclaredMethod("substring", int.class, int.class);
String object = "Hello, World!";
Object hello = substring.invoke(object, 0, 5);
System.out.println(hello); // will print out "Hello"
```

Once dynamic language resolved the reference to a `Method` object it can save it and invoke it every time it needs. One of the problems with reflection is that it is always slower than a direct invocation of a method, because JVM needs to check method visibility, receiver and arguments types, and parameters should be collected into an array (that produces "garbage"). All these checks are executed every time a method is invoked and that prevents the JVM from kicking in the JIT compiler and optimizing the code. For these and other reasons, `invokedynamic` bytecode instruction was added.

#### Invokedynamic

Despite the fact that `invokedynamic` is a bytecode instruction and most of the time is generated by compilers such as `javac` or `scalac`, Java has an API that allows application developers to use it directly. The same code from the above example will look like this:

```java
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodType methodType = MethodType.methodType(String.class, int.class, int.class);
MethodHandle substring = lookup.findVirtual(String.class, "substring", methodType); //findStatic; findSpecial
Object hello = substring.invoke("Hello, World!", 0, 5);
System.out.println(hello); // will print out "Hello"
```

When the JVM executes `MethodHandle#invoke` method the first time, it will make all checks and wire up `MethodHandle` with the exact method. Thus next  `MethodHandle#invoke` method invocations will be direct calls of a method that was looked up. At the same time, `MethodHandle`s allow us to bind particular argument values to parameters, as in the code below:

```java
MethodHandle substring = lookup.findVirtual(String.class, "substring", methodType);
MethodHandle helloWorldSubstring = substring.bind("Hello, World!");
Object hello = substring.invoke(0, 5);
System.out.println(hello); // will print out "Hello"
```

### Small experiment

Let's play around with a simple benchmark example to get a feel for how `MethodHandle` could help dynamic language runtime improve performance. The examples will be an array of random integers folding and getting a substring of `"Hello, world!"` string.

```java
public int arrayFolding() {
  int[] data = Arrays.copyOf(this.data, this.data.length);
  int sum = 0;
  for (int i : data) {
    sum += i;
  }
  return sum;
}

@Benchmark
public int arrayFoldingBaseline() {
  return arrayFolding();
}

@Benchmark
public int arrayFoldingReflectCall() throws Exception {
  return (int) arrayFoldingReflect.invoke(this);
}

@Benchmark
public int arrayFoldingMHCall() throws Throwable {
  return (int) arrayFoldingMH.invoke(this);
}

@Benchmark
public String stringConstBaseline() {
  return "Hello";
}

@Benchmark
public String substringBaseline() {
  return "Hello, World!".substring(0, 5);
}

@Benchmark
public String substringReflectCall() throws Throwable {
  return (String) substringReflect.invoke("Hello, World!", 0, 5);
}

@Benchmark
public String substringMHCall() throws Throwable {
  return (String) substringMH.invoke("Hello, World!", 0, 5);
}

@Benchmark
public String boundSubstringMHCall() throws Throwable {
  return (String) boundSubstringMH.invoke(0, 5);
}
```

The above benchmarks provide the following result on my laptop:

```sh
$ uname -a
Darwin Alexs-MacBook-Pro.local 18.2.0 Darwin Kernel Version 18.2.0: Fri Oct  5 19:41:49 PDT 2018; root:xnu-4903.221.2~2/RELEASE_X86_64 x86_64

$ java -version
openjdk version "11" 2018-09-25
OpenJDK Runtime Environment 18.9 (build 11+28)
OpenJDK 64-Bit Server VM 18.9 (build 11+28, mixed mode)

Benchmark                                           Mode  Cnt   Score   Error  Units
ReflectionVsMethodHandlers.arrayFoldingBaseline     avgt   15  17.354 ± 0.624  ns/op
ReflectionVsMethodHandlers.arrayFoldingMHCall       avgt   15  19.849 ± 0.407  ns/op
ReflectionVsMethodHandlers.arrayFoldingReflectCall  avgt   15  24.068 ± 0.497  ns/op
ReflectionVsMethodHandlers.boundSubstringMHCall     avgt   15  19.648 ± 1.604  ns/op
ReflectionVsMethodHandlers.stringConstBaseline      avgt   15   3.353 ± 0.021  ns/op
ReflectionVsMethodHandlers.substringBaseline        avgt   15   8.823 ± 0.193  ns/op
ReflectionVsMethodHandlers.substringMHCall          avgt   15  19.374 ± 0.243  ns/op
ReflectionVsMethodHandlers.substringReflectCall     avgt   15  26.658 ± 0.268  ns/op
```

As you can see, the `MethodHandle` invocation is slower than a direct call, however, it is faster than a reflection call.

You may find the full source code of the experiments [here](https://github.com/alex-dukhno/java-benchmarks/blob/master/src/main/java/org/samples/ReflectionVsMethodHandlers.java). I used [JMH](http://openjdk.java.net/projects/code-tools/jmh/) for benchmarks measurements - to run them by yourself make sure that you have [java](https://www.oracle.com/technetwork/java/javase/overview/index.html) and [maven](https://maven.apache.org) installed and simply run the following commands in a terminal:

```sh
mvn clean install
java -jar target/benchmark.jar ReflectionVsMethodHandlers
```

## InvokeConclusion

In this article, we had a brief look at different JVM bytecode instructions for method invocation and how JVM handles and executes it. We also got a taste of `invokedynamic` instruction and how it could be exploited by dynamic languages implementations on Java Virtual Machine. For those who are interested in more detailed explanations, I suggest reading [a blog post from Charles Nutter, who is a core JRuby developer](http://blog.headius.com/2008/09/first-taste-of-invokedynamic.html), and, of course, [two articles from John Rose,](https://blogs.oracle.com/jrose/method-handles-in-a-nutshell) [who is a virtual machine architect at Oracle](https://blogs.oracle.com/jrose/dynamic-invocation-in-the-vm). There are a lot of things going on inside a JVM when it handles a method invocation, and this article just scratches the surface of the complex machinery that happens under the hood of Java Virtual Machine.
