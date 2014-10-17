    Title: Creating a Language in Truffle: Part 2
    Date: 2014-10-13T17:28:12
    Tags: java, truffle, tutorial

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

    Will that function definition never change? Will a variable be a given type for the forseeable future? Use Truffle's APIs to specify this information and Graal will use it to speed up execution in the ideal case. I probably won't take advantage of many of these features, but it's another way Truffle can help you speed up your language.


Defining Mumbler Types
======================

Truffler requires you define your types so that's the first thing we'll do. Create an abstract class and list out your language's types in an annotation. Truffle will generate a concrete class that creates type check methods for all your types plus casting methods. You can override define your own check or cast methods if you choose. Since a null value in Mumbler is just an empty MumblerList we'll write a custom check function to verify that.

```java
@TypeSystem({long.class, boolean.class, MumblerFunction.class,
    MumblerSymbol.class, MumblerNull.class, MumblerList.class})
public abstract class MumblerTypes {
    @TypeCheck
    public boolean isMumblerNull(Object value) {
        return value == MumblerNull.EMPTY;
    }

    @TypeCast
    public MumblerNull asMumblerNull(Object value) {
        assert this.isMumblerNull(value);
        return MumblerNull.EMPTY;
    }
}
```

If you followed with [part 1](/blog/2014/10/13/writing-a-language-in-truffle-part-1-a-simple-slow-interpreter/) you may notice that we have new classes like `MumblerSymbol` and `MumblerNull`. Truffle separates out a languages types from its syntax nodes. This means we'll have a lot more classes in our Graal-optimized version, but if gets us a speed boost and the separation is simple and clear it's worth it.

You can also see we're using the primitive types `long` and `boolean`. Yay! No more boxing/unboxing or type checks/casts. When a node is a "number" it will really be a primitive Java long; the same for booleans. This is why separating out types from the execution nodes is so useful. Since we're not tacking on extra functionality onto numbers or booleans we can use the primitve types and get their speed.

The implementation for our custom types is simple since they don't contain anything advanced. A `MumblerSymbol` is nothing more than a name.

```java
public class MumblerSymbol {
    public final String name;

    public MumblerSymbol(String name) {
        this.name = name;
    }
}
```

Similarly, `MumblerFunction` just holds onto the top node of the function that will eventually be called. Truffle's root node is called `RootCallTarget`.

```java
public class MumblerFunction {
    public final RootCallTarget callTarget;

    public MumblerFunction(RootCallTarget callTarget) {
        this.callTarget = callTarget;
    }
}
```

`MumblerList` is a little more advanced since it implements a linked a list, but it's the same code that was in SimpleMumbler's MumblerListNode without the `eval` method.

```java
public class MumblerList {
    public final Object car;
    public final MumblerList cdr;

    protected MumblerList() {
        this.car = null;
        this.cdr = null;
    }

    private MumblerList(Object car, MumblerList cdr) {
        this.car = car;
        this.cdr = cdr;
    }

    public static MumblerList list(Object... objs) {
        return list(asList(objs));
    }

    public static MumblerList list(List<Object> objs) {
        MumblerList l = MumblerNull.EMPTY;
        for (int i=objs.size()-1; i>=0; i--) {
            l = l.cons(objs.get(i));
        }
        return l;
    }

    public MumblerList cons(Object obj) {
        return new MumblerList(obj, this);
    }

    public long length() {
        if (this == MumblerNull.EMPTY) {
            return 0;
        }

        long len = 1;
        MumblerList l = this.cdr;
        while (l != MumblerNull.EMPTY) {
            len++;
            l = l.cdr;
        }
        return len;
    }

    @Override
    public boolean equals(Object other) { /* boring */ }

    @Override
    public String toString() { /* boring */ }
}
```

We dropped the generic declarations since we won't be using this data structure to represent Mumbler programs. We won't be programming using this within Java but in our dynamically typed Mumbler language so the generic declarations would just be noise.

`MumblerNull` is just a singleton subclass of `MumblerList`. Any empty `MumblerList` must be a reference to the singleton--even the final element in all `MumblerList` lists.

```java
public final class MumblerNull extends MumblerList {

    public static final MumblerNull EMPTY = new MumblerNull();

    private MumblerNull() {}
}
```
