    Title: Creating a Language in Truffle: Part 2
    Date: 2014-10-13T17:28:12
    Tags: DRAFT

Now that I've have defined my Truffler language and have a base implementation I can start making it fast. It's time to delve into Truffler's library and use it's APIs to make Truffler faster.


Getting Started with Truffle
============================

The first thing to keep in mind is there are two main parts to Truffle: the Java APIs we call and extend which is called Truffle and the new JIT compiler added to the JVM called Graal. Graal is aware of Truffle classes and knows how to optimize the code. Truffle is still in its early stages of public availability. The documentation is sparse. The best way to learn about Truffle to watch [YouTube video tutorials](https://www.youtube.com/watch?v=N_sOxGkZfTg) and to read through the sample [SimpleLanguage source code](http://hg.openjdk.java.net/graal/graal/file/tip/graal/com.oracle.truffle.sl/src/com/oracle/truffle/sl). There are three main ways to use Truffle to speed up your interpreter.

  - Extend and use Truffle's base classes.

    Graal (the new, experimental JIT compiler for the JVM) is designed recognize Truffle classes and its subclasses. These base classes are the essense of using Truffle and Graal. Without these you're not really using Truffle and therefore not taking advantage of Graal's JIT.

    If you followed along with our simple interpreter of Truffler, you'll see analogues between Truffle's base classes and the ones created in SimpleTruffler. In many areas it'll be a simple translation to Truffle's version of base classes.

    Once we've extended Truffle's base classes Graal can start optimizing our language. It can do things like inlining code to eliminate function calls. It can also eliminate unncessary object creation in situations where it's not needed. Our simple interpreter isn't smart enough to optimize like that without much more work on our part.

  - Specify your languages types and provide type hints

    Type information is a staple of speeding up interpreters and compilers and Graal is no different. Our simple interpreter is very dumb and slow about types. Everything is a java Object; even numbers and booleans. This means that any operation on numbers and booleans need to be boxed; furthermore, type checks and casts need to be peformed before doing actual primitive calculations.

    Truffle has a way to declare your language datatypes. With this information,Truffle can create specialized eval functions (typically named "execute") so no casting, boxing and unboxing is necessary. This will be a big speed boost.

  - Add more hints on invariants

    Will that function definition never change? Will a variable be a given type for the forseeable future? Use Truffle's APIs to specify this information and Graal will you use it to speed up execution in the ideal case. I probably won't take advantage of many of these features but it's another way Truffle can help you speed up your language.
