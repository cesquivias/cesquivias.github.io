    Title: Writing a Language in Truffle: Part 1
    Date: 2014-10-13T23:25:17
    Tags: java, truffle

<p class="subtitle">How hard is it to write a simple, fast interpreter? Let's find out.</p>

Ever since implementing a [meta circular evaluator](https://mitpress.mit.edu/sicp/full-text/sicp/book/node76.html) in college I've enjoyed implementing little lisp interpreters in languages. It's one of those starting projects I do when learning a new programming language. When I first learned Python and implemented a lisp interpreter I was amazed at how small and easy to read the implementation was. Little did I know Peter Norvig beat me to [a better python lisp interpreter](http://norvig.com/lispy.html). Nonetheless, it's a fun little exercise to get acquainted with a new programming environment and appeases my curiosity in languages and interpreters.

Of course my toy interpreters are just that: a toy. Writing a lisp interpreter on top of an already slow language like Python will not win any speed competitions. You may get away with writing small Domain Specific Languages (DSLs) as an interpreter, but you can forget about any general programming language. The performance hit makes it untenable unless you write it in some lower level language like C; who wants to do that? If you want to target higher level virtual machines like the JVM you're left with writing a compiler that takes your code and produces JVM bytecode. How about writing a compiler that targets Javascript? Another not-so-fun alternative.

Thankfully, a new solution is here. You can write your interpreter in a VM that is designed to optimize your interpreter with all that wonderful JIT compilation magic. [PyPy](http://pypy.org/) has been around for a few years and has grown into such an environment. It originally started as a new Python interpreter written in a smaller, more restricted version of Python appropriately named RPython (R = restricted). You don't get all the syntactic goodness of regular Python but, it sure beats writing your interpreter in C. The developers have made herculean efforts to create a VM that can take a high level language like Python and make it fast even when written in a still high level language like RPython. It's so good it's [faster than the standard CPython](http://speed.pypy.org/) implementation in most benchmarks. It's such a promising platform there's an implementation of Ruby called [Topaz](http://topazruby.com/) that is already performing [faster in some benchmarks](http://mail.openjdk.java.net/pipermail/mlvm-dev/2013-February/005214.html) than the much more established implementations like JRuby. Exciting stuff, but now PyPy isn't the only game in town.

Oracle labs has [released its own VM](http://www.oracle.com/technetwork/oracle-labs/program-languages/overview/index-2301583.html) that hopes to achieve the same results as PyPy but that leverages the huge ecosystem of the Java Virtual Machine (JVM). This modified JVM contains a new Just-In-Time (JIT) compiler that can speed up interpreters like my little lisp to near-Java speeds. The new JIT compiler is called Graal. To take advantage of Graal's JIT-y goodness you use the Truffle library to annotate your interpreter and give Graal some hints on invariants and type information. For this integration effort you get significant speedups in your interpreter without having to resort to writing a bytecode compiler plus you have the full power of Java at your disposal.

<!-- more -->

The initial results of Truffle are very exciting. Implementations of Ruby and Javascript in Truffle have performances on the same order of magnitude as the much bigger projects of JRuby and Nashorn, respectively. They are even comparable in speed to more established projects like the Google's V8 Javascript interpreter. The kick: these Truffle implementations were done with fewer people in a shorter period of time. This means you can create your own language on the JVM that takes advantage of all it's existing libraries, native threading, JIT compiler without without having to write your own fullblown optimizing compiler, *and* you get speeds that took other languages man-years (man-decades?) to achieve. Where do I sign up?

To see if the claims are true I'm going to write another simple Lisp language which I'll call Mumbler and see how easy it is. I'll write a simple, non-Truffle interpreter and judge how easy it is to translate to Truffle. I'll take benchmarks to see what kind of speed gains we get by switching over to Truffle. If everything goes well, perhaps this proof of concept will reach speeds comparable to other production-quality lisps like Clojure, Racket or CHICKEN (scheme), but I'm getting ahead of myself.

Mumbler Language
=================

I'm going to try to keep a balance between simplicity and useful features. I don't want a language that's so simple that it's a strawman, but I don't want to spend too much time writing a language I intend to throw away. Mumbler will be a simple but fullblown language. I'll stick to basic datatypes and little-to-no syntactic sugar.

For those unfamiliar with lisps, it operates much like other dynamic languages. The biggest difference to most people is all the parentheses in the syntax, but they are there for a reason. Lisp programs are written in its own datatypes. This means that every element you see in Mumbler (and all lisps) is the same datatype you use within your own program. This uniformity gives lisps a lot of power which Mumbler sadly doesn't tap for the sake of brevity. The language will have a syntax as simple as possible and only a couple of datatypes. Lisps can go very far with what would normally be considered an anemic set of types and operations.

Datatypes
---------

  - Numbers (e.g., `1`, `400`)

    Mumbler will stick with positive integer literals. You can use the `-` function to create negative numbers if need be. This will be implemented by boxed Long. It's slow, but we're shooting for a barebones language. We also won't deal with integer overflows.

  - Boolean (i.e., `#t`, `#f`)

    Nothing special here. The tokens `#t` and `#f` will evaluate to the Java static final values `Boolean.TRUE` and `Boolean.FALSE`, respectively.

  - Symbol (e.g., `var`, `some-variable`)

    Symbols are kind of unique to lisps. If you're not familiar with them, they're string-like but are typically used for keys. Think of symbols in Ruby if you're familiar with Ruby but without the `:` in the front. Symbols show their uniqueness when they're evaluated.

  - Function (e.g., `(lambda (x) (+ x 1))`)

    Functions are written with the `lambda` special form. Functions are first class values that can be stored in a variable, passed to another function or returned by a function.

  - List (e.g., `(1 2 3)`, `(a list of symbols)`)

    Lists are the only way to make compound structures in Mumbler. It's a little limiting but it's plenty for our little language.

    Lists are implemented as singly-linked lists. There are pros and cons with this implementation, but it's the traditional implementation in lisps. We're not blazing any trails here with language design so we'll stick with the tried and true implementation.

    The empty list `()` acts as Mumblers `null` value.


Syntax
------

The syntax is simple.

  - Tokens are separated by whitespace or opening/closing parentheses.
  - Numbers are tokens that are digits. Only numbers can begin with a digit.
  - Booleans are the tokens `#t` `#f`. Any other token that begins with `#` is illegal.
  - An open paren `(` starts a new list. A `)` is closes the list. Lists can be nested.
  - Everything else is a symbol.

Here is some syntactically correct Mumbler code:

        12344
        -123
        the-above-is-a-symbol-because-Mumbler-does-not-have-negative-numbers
        #t
        (a list of symbols)

Evaluation
----------

To keep with our theme of simple, evaluation is as basic as we can make it. The program is evaluated from top to bottom. That means that a variable definition must appear before its use. Everything is an expression. Since Mumbler programs are made of Mumbler datatypes every datatype has a well-defined evaluation strategy.

  - Number (e.g., `1`, `4929`)

    Numbers evaluate to themselves. That is, a number returns a number. Shocking, I know.

  - Boolean (e.g., `#f`)

    Booleans evaluate to themselves. Same as above.

  - Symbol (e.g, `a-variable`, `a-variable-that-contains-a-function`)

    Symbols evaluate by returning looking the current namespace and return the value with the same name as the symbol. When you say `some-variable` the interpreter will see a symbol and try to evaluate it by looking in the current namespace for something stored under `"some-variable"`.

  - List (e.g., `(+ 1 2 3)`)

    Lists are evaluated as function calls. Every element in the list is evaluated before the function is applied to its arguments. The first element must evaluate to a function. The rest of the elements are the arguments. The arguments are mapped to the formal parameter names before the body of the function is called. The last expression in the function body is the return value.

Scoping
-------

Mumbler has lexical scope. If you've used any modern dynamic languages like Javascript, Python or Ruby then you know how this works. Basically, it means that functions defined within a wrapping function has access to the outer function's variables--even if the inner function is the return value and is not called until some time in the future.

Mumbler doesn't have blocks (the list of statements within `{}` in languages like Java, C, C++). New scopes are only created within new functions.

Special Forms
-------------

The evaluation process for function calls is defined to evaluate all the arguments, but we don't want this in certain circumstances. In these cases we've defined unique evaluation schemes called special forms.

  - `define`

    This creates a new variable within the current scope. `define` has to be a special form because the variable name doesn't exist yet (which is why we're calling `define`) and would throw an error for an undefined variable.

    Example: `(define c 299792458)`

  - `lambda`

    This creates an anonymous function. The second element (the first being the symbol `lambda`) is the list of formal parameters. Lambda will be used with `define` to store a function with a name. `lambda` has to be a special form because we don't want to evaluate the body immediately; only when the function is the first element in a function call. Also, the arguments need to be mapped to the parameter names before the body can be evaluated or else we would get undefined variable errors.

    The name `lambda` is used for historical purposes. We'll just say that it's the traditional name for anonymous functions in lisps and leave it at that.

    Example: `(lambda (x) (* x x))`

  - `if`

    The only control flow structure in Truffle. This works as you'd expect, the second element (the test expression) is evaluated, and if it's true the 3rd element (the "then" expression) is evaluated. If it's false the 4th element is evaluated. `if` has to be a special form because only the then or the else expression must be evaluated. If `if` was a normal function both clauses would be evaluated, the else expression would always return, and that would be bad.

    Remember, `if` is an expression like everything else. It returns the result of evaluation of either the then or else expression. Lisp ifs are more like the C-style ternary operator: `test() ? then_result() : else_result()`.

    Example: `(if (= x 0) (+ x 1) (- x 3))`

  - `quote`

    This returns the argument without evaluating it. Since all instances of Symbol and List types in Mumbler evaluate to variable lookup and function calls, respectively, we need a way to get a hold of actual Symbol and List objects. `quote` allows us to do that. With lists you can use the `list` builtin function to get a list object, but symbols need `quote` because there's no other way to get a Symbol object.

    Keep in mind, quote doesn't evaluate anything in its argument. That means if you pass a list it's elements are not evaluated. A symbol within a list will stay a symbol. You're better off using the `list` function unless you really want a list of symbols.

    Because numbers and booleans evaluate to themselves, quoting them doesn't do anything.

    Example: `(quote (a list of undefined symbols))`

With these few special forms we have a fully functional, Turing-complete programming language. We'll have to be creative in combining them to get the functionality we need, but that's part of the power of lisp.

Builtin Functions
-----------------

Although we have a Turing-complete language it isn't very useful if we can't do anything like interact with the outside world or manipulate our datatypes. Here are the builtin functions that come with Mumbler. There's nothing special about these functions other than they already exist when Mumbler starts. They are evaluated and called like user-defined functions.

  - Arimetic functions (`+`, `-`, `*`, `/`, `%`, `=`, `>`, `<`)

    We have your basic arithmetic operations. Remember, Mumbler only has integers so `/` does integer division and `%` is the modulo function.

  - List functions (`list`, `cons`, `car`, `cdr`)

    Our basic functions for creating, prepending and splicing lists. The names `cons`, `car`, `cdr` probably don't make sense if you're unfamiliar with lisp but they basically mean `prepend`, `first`, `rest` respectively. Again, we're sticking with lisp tradition here. For more background, you can read the Wikpedia article on [cons](http://en.wikipedia.org/wiki/Cons).

  - IO functions (`println`, `now`)

    Since we're not planning to do any real work in Mumbler, we'll just implement a function to send data to standard out and get a current timestamp.

I may have to add more functions in the future but this is a good start.


SimpleMumbler
==============

Now with a clear idea of what language we're creating, let's get started! We'll start with an interpreter that _doesn't_ use Truffle as a base comparison. This version will be written the way I would normally write toy langauges, à la Peter Norvig's [lispy](http://norvig.com/lispy.html).


Design
------

The architecture will be a simple pipline from program text to evaluated result.

        Read program text and return tree of expressions -> evaluate tree

The `Reader` class will encapsulate converting from program text and return the tree of Mumbler expressions. The abstract syntax tree's (AST's) top node will be called with a default environment that includes all builtin functions. When interpreting a file the result of the top evaluation will be dropped.

Our main class will look like:

```java
public class SimpleMumblerMain {
    public static void main(String[] args) throws IOException {
        assert args.length == 1 : "Mumbler file required";
        runMumbler(args[0]);
    }

    private static void runMumbler(String filename) throws IOException {
        Environment topEnv = Environment.getBaseEnvironment();

        MumblerListNode<Node> nodes = Reader.read(new FileInputStream(filename));
        for (Node node : nodes) {
            node.eval(topEnv);
        }
    }
}
```


REPL
----

We can make a small addition at essentially no cost. We can take the basic flow described above and wrap it in a `while` loop and create a [REPL](http://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop). If you've ever used languages like Python, Ruby or the Javascript console in web browsers you know a REPL. The REPL takes the text passed to it by the user and follows the flow above but then prints out the result instead of dropping it. We wrap it all in a `while` loop so we always return back to the prompt.

Our updated main class is:

```java
public class SimpleMumblerMain {
    public static void main(String[] args) throws IOException {
        assert args.length < 2 : "SimpleMumbler only accepts 1 or 0 files";
        if (args.length == 0) {
            startREPL();
        } else {
            runMumbler(args[0]);
        }
    }

    private static void startREPL() throws IOException {
        Environment topEnv = Environment.getBaseEnvironment();

        Console console = System.console();
        while (true) {
            // READ
            String data = console.readLine("~> ");
            if (data == null) {
                // EOF sent
                break;
            }
            MumblerListNode<Node> nodes = Reader.read(
                    new ByteArrayInputStream(data.getBytes()));

            // EVAL
            Object result = ListNode.EMPTY;
            for (Node node : nodes) {
                result = node.eval(topEnv);
            }

            // PRINT
            if (result != MumblerListNode.EMPTY) {
                System.out.println(result);
            }
        }
    }

    private static void runMumbler(String filename) throws IOException {
        Environment topEnv = Environment.getBaseEnvironment();

        MumblerListNode<Node> nodes = Reader.read(new FileInputStream(filename));
        for (Node node : nodes) {
            node.eval(topEnv);
        }
    }
}
```

We're being a little quick and dirty with IOExceptions but the happy path will work fine. Now time to delve into the converting an `InputStream` into a tree of Mumbler expressions.


Reading
-------

Typically, languages break this up into two stages: lexing (text -> tokens), parsing (tokens -> syntax tree). I'm going to take another page from lisp history and combine lexing and paring into one step: reading. Because we were careful in defining the syntax of Mumbler, we can easily hand-write a reader that only needs one character of look ahead to determine what token is being read.

The basic flow of the reader is to read one character then dispatch to specialized read methods depending on the character read. Our `readNode` function:

```java
public static Node readNode(PushbackReader pstream) throws IOException {
    char c = (char) pstream.read();
    pstream.unread(c);
    if (c == '(') {
        return readList(pstream);
    } else if (Character.isDigit(c)) {
        return readNumber(pstream);
    } else if (c == '#') {
        return readBoolean(pstream);
    } else if (c == ')') {
        throw new IllegalArgumentException("Unmatched close paren");
    } else {
        return readSymbol(pstream);
    }
}
```

If you look closely at our implementation of SimpleMumblerMain you'll see that we call `Reader.read` and it returns a MumblerListNode. Our `read` method can read multiple nodes at once and return them all. The `read` method:

```java
public static MumblerListNode read(InputStream istream) throws IOException {
    return read(new PushbackReader(new InputStreamReader(istream)));
}

private static MumblerListNode read(PushbackReader pstream)
        throws IOException {
    List<Node> nodes = new ArrayList<Node>();

    readWhitespace(pstream);
    char c = (char) pstream.read();
    while ((byte) c != -1) {
        pstream.unread(c);
        nodes.add(readNode(pstream));
        readWhitespace(pstream);
        c = (char) pstream.read();
    }

    return MumblerListNode.list(nodes);
}
```

Within `Reader` we only deal with `PushbackReader` objects. The public method `read(InputStream istream)` just converts the `InputStream` to a `PushbackReader`.

As you can see the `read` method is straightforward. It accumulates all Node values into a list while the end-of-file hasn't been reached then returns a MumblerListNode when it's done. It's also responsible for reading and throwing away whitespace so `readNode` doesn't have to worry about it.

I'm going to skip passed showing all the different read methods. They're what you expect: read until you find a separator then convert the string to it's respective type (i.e., number, boolean or symbol). I will show how lists are read because of some interesting twists. Lists are recursive and can read other nodes. We also check for special forms.

```java
private static Node readList(PushbackReader pstream) throws IOException {
    char paren = (char) pstream.read();
    assert paren == '(' : "Reading a list must start with '('";
    List<Node> list = new ArrayList<Node>();
    do {
        readWhitespace(pstream);
        char c = (char) pstream.read();

        if (c == ')') {
            // end of list
            break;
        } else if ((byte) c == -1) {
            throw new EOFException("EOF reached before closing of list");
        } else {
            pstream.unread(c);
            list.add(readNode(pstream));
        }
    } while (true);
    return SpecialForm.check(MumblerListNode.list(list));
}
```

The two interesting bits of this method are (1) we recursively call `readNode` to handle reading all elements in the list. Because start and end parentheses are separators we know that `readList` will get back control once they are reached; (2) We call `SpecialForm.check` to see if the list is not a function but actually one of our four special forms. The code for `SpecialForm.check` is straightforward.

```java
private static final SymbolNode DEFINE = new SymbolNode("define");
private static final SymbolNode LAMBDA = new SymbolNode("lambda");
private static final SymbolNode IF = new SymbolNode("if");
private static final SymbolNode QUOTE = new SymbolNode("quote");

public static Node check(MumblerListNode l) {
    if (l == MumblerListNode.EMPTY) {
        return l;
    } else if (l.car.equals(DEFINE)) {
        return new DefineSpecialForm(l);
    } else if (l.car.equals(LAMBDA)) {
        return new LambdaSpecialForm(l);
    } else if (l.car.equals(IF)) {
        return new IfSpecialForm(l);
    } else if (l.car.equals(QUOTE)) {
        return new QuoteSpecialForm(l);
    }
    return l;
}
```

There's nothing special about checking for special forms. I just want to make sure they aren't evaluated as function calls. We'll go into more detail about special forms later. The reader checks for special forms so the list type doesn't have to constantly check if the first element of the is a special form when evaluating. This will speed up evaluation of function calls.


Eval
----

All the nodes the reader returns extend from the abstract class `Node`. All `Node` does is define an abstract method `eval`. `eval` takes an `Environment` argumnet that contains the namespace for all the defined variables. For most datatypes (Number, Boolean, Function) `eval` returns the same object. Actually, Number and Boolean return the boxed values of `java.lang.Long` and `java.lang.Boolean`, respectively, because Java doesn't allow us to subclass `java.lang.Long` and `java.lang.Boolean`. If we could we would just return `this` in those cases. The difference is subtle and doesn't really matter. For the `Function` type, we do return `this` since we define a custom type.

```java
public abstract class Node {
    public abstract Object eval(Environment env);
}

public class Number extends Node {
    private final Long num;
    public Number(Long num) {
        this.num = num;
    }

    @Override
    public Object eval(Environment env) {
        return this.num;
    }

}

public class Function extends Node {
    // other code...

    @Override
    public Object eval(Environment env) {
        return this;
    }
}
```

`Symbol` is a little more complicated but only just. When `Symbol` is evaluated the name of the value is looked up in the `Environment` namepace and the value stored there is returned.


```java
public class SymbolNode extends Node {
    private String name;
    public SymbolNode(String name) {
        this.name = name;
    }

    @Override
    public Object eval(Environment env) {
        return env.getValue(this.name);
    }
}
```


The Environment class is just a simple mapping between strings and objects. Because Mumbler is lexically scoped, the Environment's parent is searched if the name cannot be found. If the top parent is reached and the name is never found an exception is thrown.

```java
public class Environment {
    private final HashMap<String, Object> env = new HashMap<String, Object>();

    private final Environment parent;
    public Environment() {
        this(null);
    }

    public Environment(Environment parent) {
        this.parent = parent;
    }

    public Object getValue(String name) {
        if (this.env.containsKey(name)) {
            return this.env.get(name);
        } else if (this.parent != null) {
            return this.parent.getValue(name);
        } else {
            throw new RuntimeException("No variable: " + name);
        }
    }

    public void putValue(String name, Object value) {
        this.env.put(name, value);
    }
}
```

`MumblerListNode` is evaluated as a function call. This means all elements of the list are first evaluated by calling their respective `eval` methods. The first element is then cast to a `Function` and its `apply` method is called with the rest of the list elements passed as arguments.

```java
public class MumblerListNode extends Node implements Iterable<Node> {
    // other code...

    @Override
    public Object eval(Environment env) {
        Function function = (Function) this.car.eval(env);

        List<Object> args = new ArrayList<Object>();
        for (Node node : this.cdr) {
            args.add(node.eval(env));
        }
        return function.apply(args.toArray());
    }
}
```

### Function invocation

This would be a good time to discuss what happens when user-defined functions (lambdas) are called. Remember that lambdas are really nothing more than a list of nodes so all we really need to do is evaluate the nodes in order then return the value of the final one. There are a couple of little twists. First, functions start a new namespace that have their outer function's namespace as their parent. Second, functions can have arguments so we'll need to put the arguments in the function's new namespace before we start evaluating the body lest we get incorrect unknown variable exceptions. Here's the `Function` instance that's returned when you use a lambda special form.

```java
new Function() {
    @Override
    public Object apply(Object... args) {
        Environment lambdaEnv = new Environment(parentEnv);
        if (args.length != formalParams.length()) {
            throw new RuntimeException(
                    "Wrong number of arguments. Expected: " +
                            formalParams.length() + ". Got: " +
                            args.length);
        }

        // Map parameter values to formal parameter names
        int i = 0;
        for (Node param : formalParams) {
            SymbolNode paramSymbol = (SymbolNode) param;
            lambdaEnv.putValue(paramSymbol.name, args[i]);
            i++;
        }

        // Evaluate body
        Object output = null;
        for (Node node : body) {
            output = node.eval(lambdaEnv);
        }

        return output;
    }
};
```

### Special form evaluation

We've covered how Mumbler's basic datatypes are evaluated; all that's left are the few special cases: `define`, `lambda`, `if` and `quote`. Remember, in the `Reader` when we encounter list forms that have these special symbols as their first element we replace the whole MumblerListNode object with a new `SpecialForm`. This way, when we `eval` them, we can have whatever we want happen. For example, `define` takes the third element (a value) and stores it in the current namespace under the name given in the second element.

```java
private static class DefineSpecialForm extends SpecialForm {
    public DefineSpecialForm(MumblerListNode listNode) {
        super(listNode);
    }

    @Override
    public Object eval(Environment env) {
        SymbolNode sym = (SymbolNode) this.node.cdr.car; // 2nd element
        env.putValue(sym.name,
                this.node.cdr.cdr.car.eval(env)); // 3rd element
        return MumblerListNode.EMPTY;
    }
}
```

I showed the return `Function` object for the `LambdaSpecialForm`. The rest of `LambdaSpecialForm` doesn't really do much beside getting references to the outer function's namespace and the formal parameters. Let's show the rest.

```java
private static class LambdaSpecialForm extends SpecialForm {
     public LambdaSpecialForm(MumblerListNode paramsAndBody) {
         super(paramsAndBody);
     }

     @Override
     public Object eval(final Environment parentEnv) {
         final MumblerListNode formalParams = (MumblerListNode) this.node.cdr.car;
         final MumblerListNode body = this.node.cdr.cdr;
         return new Function() { /* function definition goes here */ };
     }
 }
```

The other [`if`](https://github.com/cesquivias/mumbler/blob/master/simple/src/mumbler/simple/node/SpecialForm.java#L63) and [`quote`](https://github.com/cesquivias/mumbler/blob/master/simple/src/mumbler/simple/node/SpecialForm.java#L83) special forms do what you expect. If you're interested, you can check out the code.


### Builtin Functions

We're in the homestretch of the implementation. The only thing left is creating our builtin functions. Since we're defining the function entirely in Java we can skip things like mapping arguments and just get straight into evaluating. Most of our numerical functions can take variable arguments. For example, addition can take 0 or more arguments. With 0 arguments `0` is returned, with 1 argument the argument is returned and with more the sum of all arguments is returned.

```java
static final Function PLUS = new BuiltinFn("PLUS") {
    @Override
    public Object apply(Object... args) {
        long sum = 0;
        for (Object arg : args) {
            sum += (Long) arg;
        }
        return sum;
    }
};
```

Subtraction requires at least one argument. With one argument it negates the argument, with more it continually subtracts from the first argument.

```java
static final Function MINUS = new BuiltinFn("MINUS") {
    @Override
    public Object apply(Object... args) {
        if (args.length < 1) {
            throw new RuntimeException(this.name + " requires an argument");
        }
        switch (args.length) {
        case 1:
            return -((Long) args[0]);
        default:
            long diff = (Long) args[0];
            for (int i=1; i<args.length; i++) {
                diff -= (Long) args[i];
            }
            return diff;
        }
    }
};
```

You get the idea. You can see the implementation of the other builtin functions on [GitHub](https://github.com/cesquivias/mumbler/blob/master/simple/src/mumbler/simple/env/BuiltinFn.java).


Print
-----

Rounding out the REPL, we print out the result. There's not much here. We just call `System.out.println` and that's that. Okay, there's one little catch. Since we use the empty list as Mumbler's "null" value, we just don't print anything if that's what's returned in the REPL.


Mumbler in action
==================

Now that we have a language spec and an implementation we can take it for a spin. We would start with Hello World, but we didn't imlement strings! We'll do the next best thing and use symbols.

```scheme
(println (quote hello-world!))
; 'hello-world!
```

Close enough, and it works! Let's try something a little more involved. How about the good ol' fibonacci sequence.

```scheme
(define fibonacci
  (lambda (n)
    (if (< n 2)
        1
        (+ (fibonacci (- n 1))
           (fibonacci (- n 2))))))

(fib 10)
; 55
```
Great! We get the correct answer. We have a working program! As you can see, Mumbler would be easier to read if it had nice sugar like a function-define, but we can get by without it.


Benchmarks
==========

With our working program what kind of performance do we get? More importantly, how does it compare to other languages that have had a lot of effort put into them to make them fast? Let's take our fibonacci example and see how it performs in Racket and Node.js. The results:

        rkt
        --------------
        1346269
        computation time: 15
        total time: 117
        
        js
        --------------
        1346269
        computation time: 16
        total time: 82


That's pretty fast. How does SimpleMumbler do?
        
        mumbler
        --------------
        1346269
        ('computation-time: 1502)
        total time: 1644

Ouch. Things aren't looking so hot for our little interpreter. It performs about 100x _slower_ than Racket or Node.js. Now, both Racket and Node.js have JIT compilers and SimpleMumbler doesn't have any of that magic. What happens if we compare it to an interpreter with no JIT like CPython?

        py
        --------------
        1346269
        computation time: 385
        total time: 400

Better. SimpleMumbler is only 4x slower than CPython. Not bad for a language quicky thrown together. CPython is written in C and has a bytecode interpreter that helps explain why it's faster than our little language.

I'll settle for fibonacci as a benchmark. Our terribly inefficient algorithm gives us plenty of avenues for improvement. We make a lot of function calls so if we improve that we should get a big speed up. Getting rid of boxed longs would really help.

Algorithms > Interpreter
---------------------------------------

Out of curiosity, I converted the Mumbler implementation to one that uses a much faster, linear algorithm.

```scheme
(define fibonacci
  (lambda (n)
    (define iter
      (lambda (i n1 n2)
        (if (= i 0)
            n2
            (iter (- i 1)
                  n2
                  (+ n1 n2)))))
    (iter n 0 1)))
```
This version runs through the fibonacci numbers sequentially instead of blowing up exponentially. This is an example of [linear recursion](http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-11.html#%_sec_1.2.1) vs [tree recursion](http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-11.html#%_sec_1.2.2). You can read [SICP](http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-11.html#%_sec_1.2) for more info on this. So how does our smarter algorithm perform:

        mumbler
        --------------
        1346269
        ('computation-time: 8)
        total time: 169

Wow. Now that's an improvement. The algorithm chosen has much more to do with speed than the language implementation. In it's current state, SimpleMumbler could be "good enough" for many small tasks and give you reasonable speed. If you're writing a DSL to glue code written in other faster languages maybe a simple interpreter is all you need. But let's assume we're writing a general purpose language that needs faster function calls and numeric operations.


Next Time
=========

We have a working language and we have some baseline numbers to see if Truffle improves on our dead-simple interpreter. Next time we'll actually start looking at Truffle and begin migrating SimpleMumbler over to it. Maybe along the way we'll add some more benchmarks to get a better picture of our language's speed.
