    Title: Writing a Language in Truffle. Part 3: Making it Faster
    Date: 2015-01-04T21:49:36
    Tags: DRAFT

After the last post, we have a working interpreter in Truffle (yay!), but the results weren't very exciting. Running our fibonacci benchmark we got a paltry four and half second run time for our interpreter. Perhaps we can do better.

With help from a few couple of Truffle veterans, I was able to speed up my interpreter's speed. If you recall, the execution time for my fibonacci benchmark last time was 6.3 seconds. Not very impressive. With a few modifications and warming up the VM I was able to get the execution time down to 0.35 seconds. A 20x jump!

Let's go through the steps to get such a jump.

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc/generate-toc again -->
**Table of Contents**

- [Warming Up Graal](#warming-up-graal)
- [Improving Variable Lookup](#improving-variable-lookup)
- [Making Function Calls Faster](#making-function-calls-faster)
- [Cheating](#cheating)
    - [Hard Coding Builtin Functions](#hard-coding-builtin-functions)
- [What's Left](#whats-left)

<!-- markdown-toc end -->


<!-- more -->

Warming Up Graal
================

The first trick is the simplest. The previous versions of the benchmark didn't warmup the virtual machine properly. After a few run, things get a lot faster. How fast? Well, if we run our benchmark a few times before measuring our execution time like:

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

Off the bat, we're getting over a 6x jump. Pretty good. This means that Mumbler is already in the ballpark of some other interpreted languages. The Python version runs about 3x faster so we're almost there.

Of course this means that languages built with Truffle are better suited for longer services or at least programs that aren't expected to complete immediately. It's not the best to write a scripting langauge.

The Truffle team is working on another project that could solve this called the Substrate VM. Basically, the Graal VM get's bundled with all its optimizations into a native executable that'll start up in no time. Best of both worlds! I look forward to trying it out when it's released to the public.


Improving Variable Lookup
=========================

I mentioned at the end of Part 2 that my implementation of looking up variable values was slow. This is a prime area to look for performance improvements. There are actually a couple of ways we can approach this. (Christian Humer)[https://github.com/Grashalm] provided (one strategy)[https://github.com/cesquivias/mumbler/commit/5c448ad77a5bc7b9bad7c59247f630d4d358b3f5]. After writing my first version I was envisioning taking a different approach to speeding up variable lookup. I'll show both versions and compare the changes in execution time.

Remember Scope Depth
--------------------

Christian's approach was to cache the depth we had to traverse to get to the correct value. In case you forgot how TruffleMumbler does variable lookup, I basically check the current scope (VirtualFrame) for the variable name. If it returns null, I check it's lexical scope. If that is null I continue up the chain until we reach the end. If I reach the end I throw an exception saying there is no such variable. You can read the section on [implemented reading variables](202014-12-15-writing-a-language-in-truffle-part-2.md#node-to-read-variables) from the previous post or just [skip the code](202014-12-15-writing-a-language-in-truffle-part-2.md#specialized-read-methods).

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

We also annotate our new cache fields with `@CompilationFinal` so Truffle knows these values won't change any longer. Marking the fields with Java's `final` keyword would also work, but since we can't do that here `@CompilationFinal` is the next best thing. Be sure to call `CompilerDirectives.transferToInterpreterAndInvalidate()` before changing any value anontated with `@CompilationFinal` so Graal can throw away it's old code re-optimize with the new value.

So what kind of speed up did we see? Here's the median result after 5 runs.

<TODO />

Direct Lookup to Lexical Scope
------------------------------

Caching the depth does a good job of speeding up Mumbler so we could leave things as they are and move on, but I'm curious to see how my original idea would work.

I was thinking, why bother walking up the lexical scopes every time? Since lexical scopes are stored in the heap (as a `MaterializedFrame`) why can't we just keep a reference of the correct lexical scope and don't bother with any `for` or `while` loop. So after I go up the scope stack I'll just use Truffle's nifty tree rewriting feature and replace the slow version with a node that has a reference to the `MaterializedFrame` instance and will perform a simple (and fast) get call.

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

At this point I'll assume you're familiar with Truffle's DSL so I won't explain all the little details. I refer to my [previous post](2014-12-15-writing-a-language-in-truffle-part-2.md) if you need a referesher.

There isn't anything special here. We call the correct getter method and let Truffle deal with optimizing the tree.

In our old, slow node we just need to rewrite the tree and insert an instance of `LexicalReadNode`.

```java
    private <T> T readUpStack(FrameGet<T> getter, VirtualFrame virtualFrame)
            throws FrameSlotTypeException {
        // unchanged readUpStack code

        CompilerDirectives.transferToInterpreterAndInvalidate();
        this.replace(LexicalReadNodeFactory.create(frame, slot));
        return value;
```

I just add the two lines to switch back to the interpreter so Graal can reoptimize the code, and I insert the new `LexicalReadNode` instance.

Did my intuition pan out? Let's see what change in speed we get.

<TODO />

Making Function Calls Faster
============================

The other area with low hanging fruit was function calls. I left the implemention in v1 simple but also open to speedups. The main strategy to speeding up function calls is to switch from using `IndirectCallNode` to using `DirectCallNode`. Graal can make more optimization with `DirectCallNode` so we should switch over if we can.

Many languages have more complicated calling semantics because of competing scopes (current block, outer block, class instance, class static, parent class instance, parent class static, ...) but thankfully Mumbler doesn't have to deal with any of that. When Mumbler resolves a function there's only value it can be (the first value returned by climbing the scope stack), and that won't change. Because of this, we can switch to `DirectCallNode` without having to worry too much about rolling back our optimization if our assumption becomes false. If the variable lookup returns a different function for some reason we'll just throw an errorfor now.


Cheating
========


Hard Coding Builtin Functions
-----------------------------


What's Left
===========

