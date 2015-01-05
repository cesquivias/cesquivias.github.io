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


Making Function Calls Faster
============================


Cheating
========


Hard Coding Builtin Functions
-----------------------------


What's Left
===========

