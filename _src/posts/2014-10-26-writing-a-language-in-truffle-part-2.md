    Title: Writing a Language in Truffle: Part 2
    Date: 2014-10-13T17:28:12
    Tags: java, truffle, tutorial

Now that I've have defined my Truffler language and have a base implementation I can start making it fast. It's time to delve into Truffler's library and use it's APIs to make Truffler faster.


Getting Started with Truffle
============================

The first thing to keep in mind is there are two main parts to Truffle: the Java APIs we call and extend which is called Truffle and the new JIT compiler added to the JVM called Graal. Graal is aware of Truffle classes and knows how to optimize the code. Truffle is still in its early stages of public availability. The documentation is sparse. The best way to learn about Truffle to watch [YouTube video tutorials](https://www.youtube.com/watch?v=N_sOxGkZfTg) and to read through the sample [SimpleLanguage source code](http://hg.openjdk.java.net/graal/graal/file/tip/graal/com.oracle.truffle.sl/src/com/oracle/truffle/sl). There are three main ways to use Truffle to speed up your interpreter.

  - Extend and use Truffle's base classes.

    Graal (the new, experimental JIT compiler for the JVM) is designed recognize Truffle classes and its subclasses. These base classes are the essense of using Truffle and Graal. Without these you're not really using Truffle and therefore not taking advantage of Graal's JIT.

    If you followed along with our simple interpreter of Truffler, you'll see analogues between Truffle's base classes and the ones created in SimpleMumbler. In many areas it'll be a simple translation to Truffle's version of base classes.

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

With Mumber types defined and a `MumberTypes` class created Truffle now generates a `MumberTypesGen` class with all the type check/casting methods we'll need.


Create Syntax Tree Using Truffle
====================================

Mumbler now has its types defined. Let's now define the nodes that will define its expressions.

Defining the Base Node Class
----------------------------

In SimpleMumbler we created a base abstract class called `Node` that was extended by all the expression nodes the `Reader` created. Truffle has a base class you need to use for classes that make up your syntax tree: [`Node`](http://hg.openjdk.java.net/graal/graal/file/tip/graal/com.oracle.truffle.api/src/com/oracle/truffle/api/nodes/Node.java). Truffle still wants you to create your own root class that extends `Node` and will be extended by all your syntax nodes.

```java
public abstract class MumblerNode extends Node {
    public abstract Object execute(VirtualFrame virtualFrame);
}
```

`MumblerNode` will be the parent class for all nodes the reader will create for Mumbler. The method that all children must implement is called `execute`. We called the generic method `eval` in SimpleMumbler but it's typically called `execute` in Truffle so we'll stick with that practice.

If we left `MumblerNode` like this things could work, but it would mean that every expression would have to return an `Object` instance. There's no place for primitive types to find a way through. We're back to having to use boxing/unboxing and casting. This is obviously unideal. We would like to have equivalent `execute` methods but with primitive return types. Unfortunately, Java doesn't have a way to differentiate methods by return type. Truffle has a way around this. We can create special `execute*` methods for every type our language defines. If we then annotate `MumblerNode` with `TypeSystemReference` and point it to `MumblerTypes` Truffle will see we have an execute method for each type and inline the faster method call. After creating special execute methods for each Mumbler type we get:

```java

@TypeSystemReference(MumblerTypes.class)
@NodeInfo(description = "The abstract base node for all expressions")
public abstract class MumblerNode extends Node {

    public abstract Object execute(VirtualFrame virtualFrame);

    public long executeLong(VirtualFrame virtualFame)
            throws UnexpectedResultException {
        return MumblerTypesGen.MUMBLERTYPES.expectLong(
                this.execute(virtualFame));
    }

    public boolean executeBoolean(VirtualFrame virtualFrame)
            throws UnexpectedResultException {
        return MumblerTypesGen.MUMBLERTYPES.expectBoolean(
                this.execute(virtualFrame));
    }

    public MumblerSymbol executeMumblerSymbol(VirtualFrame virtualFrame)
            throws UnexpectedResultException {
        return MumblerTypesGen.MUMBLERTYPES.expectMumblerSymbol(
                this.execute(virtualFrame));
    }

    public MumblerFunction executeMumblerFunction(VirtualFrame virtualFrame)
            throws UnexpectedResultException {
        return MumblerTypesGen.MUMBLERTYPES.expectMumblerFunction(
                this.execute(virtualFrame));
    }

    public MumblerNull executeMumblerNull(VirtualFrame virtualFrame)
            throws UnexpectedResultException {
        return MumblerTypesGen.MUMBLERTYPES.expectMumblerNull(
                this.execute(virtualFrame));
    }
}
```

The specialized `MumblerNode` class has a lot more lines than the old one, but it's all pretty boilerplate. What it's basically saying is call the generic `execute` method and verify the return value is the type you expect. If the type is wrong, throw an UnexpectedResultException. If Graal knows a node is always returning a certain type (a `long` for example) then it will skip the slow, generic method and call the faster `executeLong` method. You'll have to define the faster `executeLong` method in your Node subclasses, but Truffle and Graal take care of the plumbing for you. Graal will throw away its optimized code and fallback on the generic `execute` method if the types change. You always get the correct result but now Graal can optimize primitive return types with a few extra lines of code.


Simple Literal Nodes
--------------------

Now that we have a base expression node let's implement some concrete nodes. The simplest would be literal nodes for numbers and booleans.

```java
public class NumberNode extends MumblerNode {

    private final long num;

    public NumberNode(long num) {
        this.num = num;
    }

    @Override
    public Object execute(VirtualFrame virtualFrame) {
        return this.num;
    }

    @Override
    public long executeLong(VirtualFrame virtualFrame) {
        return this.num;
    }
}
```

GraalMumbler's `NumberNode` is very similar to the SimpleMumbler's. We store the value of the number and return it when it's evaluated/executed (we'll call it "executed" in the future to be consistent). The only twist is the additional `executeLong` method. When Graal discovers that a NumberNode instance always returns a `long` (which should be quickly) it will start to use the unboxed `executeLong`. We can do the same with the `BooleanNode`.

```java
public class BooleanNode extends MumblerNode {

    private final boolean bool;

    public BooleanNode(boolean bool) {
        this.bool = bool;
    }

    @Override
    public Object execute(VirtualFrame virtualFrame) {
        return this.bool;
    }

    @Override
    public boolean executeBoolean(VirtualFrame virtualFrame) {
        return this.bool;
    }
}
```

We now have valid nodes that Truffle can execute and speed up. If our language was nothing but number and boolean literals we'd be set, but we still have a ways to go. Let's move up the complexity ladder and try to implement a node that doesn't return a constant value.


SymbolNode - Reading Variables
------------------------------

The next most complex node is the `SymbolNode`. When executed, it looks in the current namespace (called a [`Frame`](http://hg.openjdk.java.net/graal/graal/file/tip/graal/com.oracle.truffle.api/src/com/oracle/truffle/api/frame/Frame.java) in Truffle) and returns the value stored. In SimpleMumbler we used strings as keys. Truffle requires a more advanced object. It's nothing too complicated. It's called a `FrameSlot`, and along with having an ID to distinguish one variable from another it stores other metadata. It stores type information in something called `FrameSlotKind`. `FrameSlotKind` separates out primitive types from objects. There are other metadata in `FrameSlot` but we won't use those.

Let's write a version of `SymbolNode` that pulls the value from a `Frame`.

```java
public class SymbolNode extends MumblerNode {
    private final FrameSlot slot;

    public SymbolNode(FrameSlot slot) {
        this.slot = slot;
    }

    @Override
    public Object execute(VirtualFrame virtualFrame) {
        return virtualFrame.getValue(this.slot);
    }
}
```

This version will get the job done. The reader will pass a `FrameSlot` and that will be used to find the value in the current frame. It make work but as you've seen before we need to do a little more for primitive types. We'll use the Truffle DSL to most of the plumbing. We just need to define our specialized methods for reading values of different types.

```java

@NodeField(name = "slot", type = FrameSlot.class)
public abstract class SymbolNode extends MumblerNode {
    protected abstract FrameSlot getSlot();

    @Specialization(rewriteOn = FrameSlotTypeException.class)
    protected long readLong(VirtualFrame virtualFrame)
            throws FrameSlotTypeException {
        return virtualFrame.getLong(this.getSlot());
    }

    @Specialization(rewriteOn = FrameSlotTypeException.class)
    protected boolean readBoolean(VirtualFrame virtualFrame)
            throws FrameSlotTypeException {
        return virtualFrame.getBoolean(this.getSlot());
    }

    @Specialization(rewriteOn = FrameSlotTypeException.class)
    protected Object readObject(VirtualFrame virtualFrame)
            throws FrameSlotTypeException {
        return virtualFrame.getObject(this.getSlot());
    }

    @Specialization(contains = {"readLong", "readBoolean", "readObject"})
    protected Object read(VirtualFrame virtualFrame) {
        return virtualFrame.getValue(this.getSlot());
    }
}
```

This version has gotten a lot longer but the new pieces are pretty straightforward. First, `execute` is no longer defined. The `SymbolNode` has become abstract and defines `read` methods instead. The Truffle DSL will generate a concrete subclasses of `SymbolNode` that will define `execute` and call our `read` methods.

The `read` method is our new generic method that will always return a value--even boxed primitives. That method will be called and after Graal determines the type of the value it can then start to call our new specialized methods `readLong`, `readBoolean` and `readObject`. `readObject` retuns an `Object` like our generic `read` but this method is expected to *only* return objects; primitives will not be boxed and returned from `readObject`. The `@Specialization` annotation on `read` tells Graal what our specialized methods are. The `@Specialization` annotation on our other methods tell Graal to stop calling this method if a `FrameSlotTypeException` is thrown while reading. If our `Frame` sees we're trying to read out a `long` but a `boolean` is stored it will throw a `FrameSlotTypeException`.

Lastly, we dropped our constructor. We're letting the DSL take care of generating the constructor, but we need to tell it we need a field defined. The `NodeField` annotation takes care of that for us and the abstract `getSlot` method will be implemented and give us our `FrameSlot`.

DefineNode - Writing Variables
------------------------------

I'm going to skip ahead and show the `define` special form implementation. `define` is how set variable values in Mumbler so it bookends nicely with `SymbolNode`. I'll skip the really simple version of `DefineNode` since it's basically the same as `SymbolNode` but calls `setObject` instead of `getValue`. Again, this version is slow since all primitive values become boxed. Let's jump to the version that has all the nice specialized methods.

```java

@NodeChild("valueNode")
@NodeField(name = "slot", type = FrameSlot.class)
public abstract class DefineNode extends MumblerNode {
    protected abstract FrameSlot getSlot();

    @Specialization(guards = "isLongKind")
    protected long writeLong(VirtualFrame virtualFrame, long value) {
        virtualFrame.setLong(this.getSlot(), value);
        return value;
    }

    @Specialization(guards = "isBooleanKind")
    protected boolean writeBoolean(VirtualFrame virtualFrame, boolean value) {
        virtualFrame.setBoolean(this.getSlot(), value);
        return value;
    }

    @Specialization(contains = {"writeLong", "writeBoolean"})
    protected Object write(VirtualFrame virtualFrame, Object value) {
        FrameSlot slot = this.getSlot();
        if (slot.getKind() != FrameSlotKind.Object) {
            CompilerDirectives.transferToInterpreterAndInvalidate();
            slot.setKind(FrameSlotKind.Object);
        }
        virtualFrame.setObject(slot, value);
        return value;
    }

    protected boolean isLongKind() {
        return this.getSlot().getKind() == FrameSlotKind.Long;
    }

    protected boolean isBooleanKind() {
        return this.getSlot().getKind() == FrameSlotKind.Boolean;
    }
}
```

We'll start at the top. Again, I'm using the Truffle DSL to generate the subclasses and take care of all the defining all the plumbing connecting the generic method to the specialized. We again tell the Truffle DSL we have a `FrameSlot` field using the `@NodeField` annotation, but this time we also have a `MumblerNode` that provides the value to be stored. We use the `@NodeChild` annotation fittingly to state the `DefineNode` has a child node. We call this child node "valueNode". The DSL will execute the node and then pass value into our specialized methods.

The `writeLong` and `writeBoolean` methods are similar to what we have in `SymbolNode`, but there's one little twist. We define the `guards` to check the `FrameSlotKind` before calling a specialized method. If the guard returns `true` then the specialized method is called. If not, the next one is tried. If none return `true` the generic `write` method is called.

The `write` method acts as both generic method that will aways succeed and the method called when storing objects. Everything coming through `write` will be an object (since it failed all the previous guards). That's why we check the `FrameSlotKind` and throw away any specialized code if we had a primitive type.


InvocationNode - Executing Functions Call
-----------------------------------------

We can execute some literal values and store/fetch some variables. The next big piece of the puzzle is calling functions. For the moment, we'll assume we have some functions already defined. This code will work both built-in functions and user-defined ones.

The steps are the same in our Graal version as they were in SimpleMumbler, but Truffle requires we few class integration. Truffle also has a lot more options for speeding up function calls. There's a lot more code here than we had in SimpleMumbler, but this will be another area where we hope to see big speed boosts.

To refresh, the basic steps in making a function call are: Execute all arguments; execute the root function node with the argument values. Let's this simple version.

```java
public class InvocationNode extends MumblerNode {
    @Child private MumblerNode functionNode;
    @Children private final MumblerNode[] argumentsNodes;
    @Child IndirectCallNode callNode;

    public InvocationNode(MumblerNode functionNode,
            MumblerNode[] argumentNodes) {
        this.functionNode = functionNode;
        this.argumentsNodes = argumentNodes;
        this.callNode = Truffle.getRuntime().createIndirectCallNode();
    }

    @Override
    @ExplodeLoop
    public Object execute(VirtualFrame virtualFrame) {
        MumblerFunction function = this.getFunction(virtualFrame);

        Object[] arguments = new Object[this.argumentsNodes.length];
        for (int i=0; i<arguments.length; i++) {
            arguments[i] = this.argumentsNodes[i].execute(virtualFrame);
        }

        return this.callNode.call(virtualFrame, function.callTarget, arguments);
    }

    private MumblerFunction getFunction(VirtualFrame virtualFrame) {
        try {
            return this.functionNode.executeMumblerFunction(virtualFrame);
        } catch (UnexpectedResultException e) {
            throw new UnsupportedSpecializationException(this,
                    new Node[] {this.functionNode}, e);
        }
    }
}
```

The `execute` method contains the basic flow. We get the function object, call `execute` on our arguments then call the function with our arguments. In SimpleMumbler we just cast the first value to a function. Truffle has a more sophisticated approach. Since we have specialized `execute` methods for all out data types we can just call `executeMumblerFunction` to get our function. If the type turns out to be something other than `MumblerFunction` we throw an exception like before, but this exception will be a little more specific than our old `ClassCastException`. Getting our argument values is straightforward. It's basically what we did in SimpleMumbler.

Executing the function body requires a little more Truffle integration. We have an instance of `IndirectCallNode` which connects our function's AST to Graal's runtime. An `IndirectCallNode` is used because the function may change between calls (e.g., a variable may be redefined and a symbol would return a different function). We pull out the `RootCallTarget` from our function type which is the root node of our function's AST.

The final bit of Truffle integration is the addition of annotations to `InvocationNode`'s fields. These annotations tell Truffle that at this node (the `InvocationNode`) in the AST these are its children nodes. The `@Child` annotation is for defining one field as a child and the `@Children` annotation is for an array of nodes that are children. We haven't seen these before because our literal nodes had no children; they just returned constant values. `SymbolNode` and `DefineNode` used Truffle's DSL library which took care of defining the children for us. Here, we have to do it manually. We also annotate `execute` with `@ExplodeLoop`. This is necessary whenever you iterate through an array of nodes you annotated with `@Children` to execute.


### Making function calls faster

The version will work well but it can be faster. The main slowdown is that we have to fetch the function value on every function call. It is unlikely a function will be redefined frequently between calls so why not cache our fetched function and only refetch if something has changed. I'm going to skip the cache for now. If our GraalMumbler implementation is still very slow I'll come back and revisit making function calls faster.

If you're interested in seeing an implemention of an inline cache for function calls you can read the reference [Simple Language](hg.openjdk.java.net/graal/graal/file/tip/graal/com.oracle.truffle.sl/src/com/oracle/truffle/sl/nodes/call) Truffle provides.


More Special Forms
------------------

We have our basic datatypes defined, nodes for number and boolean literals, nodes for getting and setting variables and a node for calling a function. We almost have all the nodes we need for a fully functional language, but we still need to round out our special forms. We already defined `DefineNode` so we just need nodes for `if`, `lambda` and `quote`. We'll tackle them in that order.


### `IfNode`

`if` doesn't have much going in it so it provides a nice illustration of how Truffle influences how we write nodes.

```java
public class IfNode extends MumblerNode {
    @Child private MumblerNode testNode;
    @Child private MumblerNode thenNode;
    @Child private MumblerNode elseNode;

    private final ConditionProfile conditionProfile =
            ConditionProfile.createBinaryProfile();

    public IfNode(MumblerNode testNode, MumblerNode thenNode,
            MumblerNode elseNode) {
        this.testNode = testNode;
        this.thenNode = thenNode;
        this.elseNode = elseNode;
    }

    @Override
    public Object execute(VirtualFrame virtualFrame) {
        if (this.conditionProfile.profile(this.testResult(virtualFrame))) {
            return this.thenNode.execute(virtualFrame);
        } else {
            return this.elseNode.execute(virtualFrame);
        }
    }

    private boolean testResult(VirtualFrame virtualFrame) {
        try {
            return this.testNode.executeBoolean(virtualFrame);
        } catch (UnexpectedResultException e) {
            Object result = this.testNode.execute(virtualFrame);
            // All objects are truthy except the empty list.
            return result != MumblerNull.EMPTY;
        }
    }
}
```

Starting in `execute` again we see the simplicity of `IfNode`: it's just an `if` statement, unsurprisingly. The `testResult` method shows a little twist in Mumbler not present in Java. In Mumbler, two values are considered false; everything else is considered true. We optimistically assume the test expression will return a value of type boolean. When it does we pass that along undisturbed. But if is polymorphic. We need to be able to handle non-boolean types. The only non-boolean type we really care about is the empty list `()`. If that's returned then we know the result is `false`. We don't care about any other object. All other objects are `true`.

If has 3 children nodes so we need to annotate those with `@Child`. Since we have a well-defined number of nodes we don't need `@Children` nor do we need to annotation `execute` with `@ExplodeLoop`.

The last bit of Truffle magic added is the `ConditionProfile` object. Truffle can keep track of `true` and `false` results and speculate if a test condition always returns `true` or always returns `false`. This allows Graal to generate better code when one path always taken. The `ConditionProfile` field must be `final`.


### LambdaNode
