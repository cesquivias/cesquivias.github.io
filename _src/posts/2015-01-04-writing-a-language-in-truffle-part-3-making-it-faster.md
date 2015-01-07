    Title: Writing a Language in Truffle. Part 3: Making it Faster
    Date: 2015-01-04T21:49:36
    Tags: java, truffle, tutorial

After the last post, we have a working interpreter in Truffle (yay!), but the results weren't very exciting. Running our fibonacci benchmark we got a paltry 6.3 seconds execution time using TruffleMumbler. Perhaps we can do better.

With help from a couple of Truffle veterans, I was able to speed up my interpreter's speed. With a couple of key improvements and warming up the VM I was able to get the execution time down to 0.1 seconds. A 63x jump!

Let's go through the changes I made to get such an improvement.


<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc/generate-toc again -->
**Table of Contents**

- [Warming Up Graal](#warming-up-graal)
- [Improving Variable Lookup](#improving-variable-lookup)
    - [Remember Scope Depth](#remember-scope-depth)
    - [Direct Lookup to Lexical Scope](#direct-lookup-to-lexical-scope)
- [Making Function Calls Faster](#making-function-calls-faster)
- [Conclusion](#conclusion)

<!-- markdown-toc end -->

<!-- more -->

Warming Up Graal
================

The first change is the simplest. The previous versions of the benchmark didn't warmup the virtual machine properly. After a few runs, things get a lot faster. How fast? Well, if we run our benchmark a few times before measuring our execution time like:

```scheme
(fibonacci 30)
(fibonacci 30)
(fibonacci 30)
(fibonacci 30)
(fibonacci 30)
(fibonacci 30)
(fibonacci 30)

(define start (now))
(println (fibonacci 30))
(define end (now))

(println (list (quote computation-time:) (- end start)))
```

We go from

        mumbler (truffle)
        --------------
        1346269
        ('computation-time: 6348)


to

        mumbler (truffle)
        --------------
        1346269
        ('computation-time: 935)

Off the bat, we're getting over a 6x jump. Pretty good. This means that the Mumbler interpreter is already in the ballpark of some other interpreted languages. The Python version runs about 3x faster on CPython. We're almost there.

We probably didn't warm up the VM in the Simple Language either. Let's rerun that benchmark with some previous calls and see what kind of numbers we get.

        simple language
        --------------
        == running on Graal Truffle Runtime
        1346269
        computation time: 95

I guess there's still a lot of improvements we can make. Simple Language is 10x faster than Mumbler right now, and Mumbler is as simple (if not simpler) than Simple Language so there's no reason we shouldn't be within the same order of magnitude. We'll see where Mumbler's speed lands after our changes.

It's no surprise, like other VMs with JITs, Graal benefits when you give it time to warm up. Graal still has a hefty startup time though. Of course this means that languages built with Truffle are better suited for longer services or at least programs that aren't expected to complete immediately. That is, it's not the best platform on which to write a scripting langauge.

The Truffle team is working on another project that could solve this problem called the Substrate VM. Basically, the Graal VM gets bundled with all its optimizations into a native executable that'll start up in no time. Best of both worlds! I look forward to trying it out when it's released to the public.

For the following improvements we'll use the new benchmark script so we can be sure JIT compiler has kicked in.


Improving Variable Lookup
=========================

I mentioned at the end of Part 2 that my implementation of looking up variable values was slow. This is a prime area to look for performance improvements. There are actually a couple of ways we can approach this. [Christian Humer](https://github.com/Grashalm) provided [one strategy](https://github.com/cesquivias/mumbler/commit/5c448ad77a5bc7b9bad7c59247f630d4d358b3f5). After writing my first version I was envisioning taking a different approach to speeding up variable lookup. I'll show both versions and compare the changes in execution time.

Remember Scope Depth
--------------------

Christian's approach was to cache the depth we had to traverse to get to the correct value. In case you forgot how TruffleMumbler does variable lookup, I basically check the current scope (VirtualFrame) for the variable name. If it returns `null`, I check its lexical scope. If that is `null` I continue up the chain until we reach the end. If I reach the end I throw an exception saying there is no such variable. You can read the section on [reading variables](http://cesquivias.github.io/blog/2014/12/02/writing-a-language-in-truffle-part-2-using-truffle-and-graal/#node-to-read-variables) from the previous post or just [skip the code](http://cesquivias.github.io/blog/2014/12/02/writing-a-language-in-truffle-part-2-using-truffle-and-graal/#specialized-read-methods).

The changes we make to cache the depth are pretty simple. Our lookup method now checks to see if we previously resolved the variable. If so, we take the fast path. If not, we call the old code with the extra step of logging the depth and storing it in a field.

```java
@NodeField(name = "slot", type = FrameSlot.class)
public abstract class SymbolNode extends MumblerNode {

    @CompilationFinal
    FrameSlot resolvedSlot;
    @CompilationFinal
    int lookupDepth;

    @ExplodeLoop
    public <T> T readUpStack(FrameGet<T> getter, Frame frame)
            throws FrameSlotTypeException {
        if (this.resolvedSlot == null) {
            CompilerDirectives.transferToInterpreterAndInvalidate();
            this.resolveSlot(getter, frame);
        }

        Frame lookupFrame = frame;
        for (int i = 0; i < this.lookupDepth; i++) {
            lookupFrame = this.getLexicalScope(lookupFrame);
        }
        return getter.get(lookupFrame, this.resolvedSlot);
    }

    private <T> void resolveSlot(FrameGet<T> getter, Frame frame)
            throws FrameSlotTypeException {
        FrameSlot slot = this.getSlot();
        int depth = 0;
        Object identifier = slot.getIdentifier();
        T value = getter.get(frame, slot);
        while (value == null) {
            depth++;
            frame = this.getLexicalScope(frame);
            if (frame == null) {
                CompilerDirectives.transferToInterpreterAndInvalidate();
                throw new RuntimeException("Unknown variable: "
                        + this.getSlot().getIdentifier());
            }
            FrameDescriptor desc = frame.getFrameDescriptor();
            slot = desc.findFrameSlot(identifier);
            if (slot != null) {
                value = getter.get(frame, slot);
            }
        }
        this.lookupDepth = depth;
        this.resolvedSlot = slot;
    }

    //  more code..
}
```

Because we have a `for` loop in `readUpStack` that will have a constant number of iterations we can annotate the method with `@ExplodeLoop` like we did in previous situations further speeding up the code once it hits the JIT compiler.

We also annotate our new cache fields with `@CompilationFinal` so Truffle knows these values won't change any longer. Marking the fields with Java's `final` keyword would also work, but since we can't do that here `@CompilationFinal` is the next best thing. Be sure to call `CompilerDirectives.transferToInterpreterAndInvalidate()` before changing any value anontated with `@CompilationFinal` so Graal can throw away its old code re-optimize with the new value.

So what kind of speed up did we see? Here's the median result after 5 runs.

        mumbler (truffle)
        --------------
        1346269
        ('computation-time: 264)

Nice. An additional 3x speedup on our benchmark! I guess variable lookup was really slow. With this one change Mumbler is now faster than CPython! How exciting. Mumbler still has a way to go to compete with the Javascript V8 engine (16ms) or a JITed lisp implementation like Racket (15ms), but look at how much less work it took to implement Mumbler.

With these great results let's see if my original idea will be as successful.


Direct Lookup to Lexical Scope
------------------------------

Caching the depth does a good job of speeding up Mumbler so we could leave things as they are and move on, but I'm curious to see how my original idea would work.

I was thinking, why bother walking up the lexical scopes every time? Since lexical scopes are stored in the heap (as a `MaterializedFrame`) why can't we just keep a reference to the correct lexical scope and not bother with any `for` or `while` loops. So after I go up the scope stack I'll just use Truffle's nifty tree rewriting feature and replace the slow version with a node that has a reference to the `MaterializedFrame` instance and will perform a simple (and fast) get call.

Our new node will be simple since all it does is a get call. We just need to have a reference to the lexical scope and the variable's key into the map.

```java
@NodeFields(value={
        @NodeField(name = "scope", type = MaterializedFrame.class),
        @NodeField(name = "slot", type = FrameSlot.class)
})
public abstract class LexicalReadNode extends MumblerNode {
    protected abstract MaterializedFrame getScope();
    protected abstract FrameSlot getSlot();

    @Specialization(rewriteOn = FrameSlotTypeException.class)
    protected long readLong(VirtualFrame virtualFrame)
            throws FrameSlotTypeException {
        return this.getScope().getLong(this.getSlot());
    }

    @Specialization(rewriteOn = FrameSlotTypeException.class)
    protected boolean readBoolean(VirtualFrame virtualFrame)
            throws FrameSlotTypeException {
        return this.getScope().getBoolean(this.getSlot());
    }

    @Specialization(rewriteOn = FrameSlotTypeException.class)
    protected Object readObject(VirtualFrame virtualFrame)
            throws FrameSlotTypeException {
        return this.getScope().getObject(this.getSlot());
    }

    @Specialization(contains = {"readLong", "readBoolean", "readObject"})
    public Object read(VirtualFrame virtualFrame) {
        return this.getScope().getValue(this.getSlot());
    }
}
```

At this point I'll assume you're familiar with Truffle's DSL so I won't explain all the little details. I refer to my [previous post](http://cesquivias.github.io/blog/2014/12/02/writing-a-language-in-truffle-part-2-using-truffle-and-graal/) if you need a referesher.

There isn't anything special here. We call the correct getter method and let Truffle deal with optimizing the tree.

In our old, slow node we just need to rewrite the tree and insert an instance of `LexicalReadNode`.

```java
private <T> T readUpStack(FrameGet<T> getter, VirtualFrame virtualFrame)
        throws FrameSlotTypeException {
    // unchanged readUpStack code

    CompilerDirectives.transferToInterpreterAndInvalidate();
    this.replace(LexicalReadNodeFactory.create(frame, slot));
    return value;
}
```

I just add the two lines to switch back to the interpreter so Graal can reoptimize the code, and I insert the new `LexicalReadNode` instance.

Did my intuition pan out? Let's see what change in speed we get.

        mumbler (truffle)
        --------------
        1346269
        ('computation-time: 140)

Wow. I wasn't expecting that. Not only is my node rewriting approach fast (whew), but it's nearly twice as fast as the caching the scope depth and traversing it on every read. In all, we have an almost 7x improvement. Now getting to Racket/Node speeds doesn't seem so impossible.

I think this one change would be pretty good, but let's see what kind of gains we could get from improving function calls.


Making Function Calls Faster
============================

The other area with low hanging fruit is function calls. I left the implemention in v1 simple but also open to speedups. The main strategy to speeding up function calls is to switch from using `IndirectCallNode` to using `DirectCallNode`. Graal can make more optimization with `DirectCallNode` so we should switch over if we can.

Many languages have more complicated calling semantics because of competing scopes (current block, outer block, class instance, class static, parent class instance, parent class static, ...) but thankfully Mumbler doesn't have to deal with any of that. When Mumbler resolves a function there's only value it can be (the first value returned by climbing the scope stack), and that won't change. Because of this, we can switch to `DirectCallNode` without having to worry too much about rolling back our optimization if our assumption becomes false. If the variable lookup returns a different function for some reason we'll just throw an error for now.

The only changes necessary are in the `execute` method of the `InvokeNode` class.

```java
    @Override
    @ExplodeLoop
    public Object execute(VirtualFrame virtualFrame) {
        MumblerFunction function = this.evaluateFunction(virtualFrame);
        CompilerAsserts.compilationConstant(this.argumentNodes.length);

        Object[] argumentValues = new Object[this.argumentNodes.length + 1];
        argumentValues[0] = function.getLexicalScope();
        for (int i=0; i<this.argumentNodes.length; i++) {
            argumentValues[i+1] = this.argumentNodes[i].execute(virtualFrame);
        }

        if (this.callNode == null)  {
            CompilerDirectives.transferToInterpreterAndInvalidate();
            this.callNode = this.insert(Truffle.getRuntime().createDirectCallNode(function.callTarget));
        }

        if (function.callTarget != this.callNode.getCallTarget()) {
            CompilerDirectives.transferToInterpreterAndInvalidate();
            throw new UnsupportedOperationException("need to implement a proper inline cache.");
        }

        return this.callNode.call(virtualFrame, argumentValues);
    }
```

The method begins like it used to. We get the function then evaluate the arguments before proceeding to making the actual call. The main difference here is we check if `callNode` is set. If not we create a direct call node before proceeding. Because we need the function's call target value, we have to do this at runtime because we don't have that information when reading in the code.

Once we have the `callNode` set we make sure the call target of `callNode` and the function's are the same. The two should be the same since Mumbler has no way to redefine a function's call target (it's actually set as `final`), but just to be a little defensive we throw an exception if they are ever different.

Finally we make our function call with our direct call node.

So what kind of speed improvements, if any, to we see with this change? First let's run the benchmark without the changes we made to variable lookup above. After that, we'll see the overall speed change with all accumulated changes.

Only function call changes:

        mumbler (truffle)
        --------------
        ('computation-time: 857)
        1346269

That's a respectable improvement. We dropped almost 100ms. We're getting about an 8% improvement. Not as stupendous as our variable lookup but I'll take 8% for such little work any day of the week.

Let's see what kind of numbers we get with this change plus the changes to variable lookup.

        mumbler (truffle)
        --------------
        1346269
        ('computation-time: 98)

Mumbler broke 100ms! It's arbitrary but it's still a nice little victory. With these two changes Mumbler is almost 10x faster on a warmed up VM. With two simple improvements we've reached Simple Language speeds. I don't think you can ask for much more than that.


Conclusion
==========

Though Mumbler's speed has come a long way with these couple of changes I did learn quite a bit about where Graal spends its time interpreting Mumbler. I had no idea traversing the lexical scopes would take so much effort. I guess it shouldn't be surprising. Taking a quick glance at the fibonacci benchmark, there are 25 symbols resolved at runtime. 10 `fibonacci`, 3 `n`, 3 `-`, 2 `now`, 2 `println`, 1, `<`, 1, `+`, 1 `list`, 1 `start`, 1 `end`. Inside the main fibonnaci function it only does 4 arithmetic operations, but performs 9 variable lookups!

Though function calls don't happen as often as variable lookup it does play a key role in Mumbler. Like most other lisps, there aren't any loop constructs like `for` or `while`. You're expected to use recursion or higher order functions that use recursion underneath the covers. Making function calls as fast as possible would make sure Mumbler stays a fast language.

I don't think you could ask for a better result. Truffle may not (yet) be the ideal platform for writing a scripting language, but the VM has proven to produce great performance for relatively little effort on the implementer.
