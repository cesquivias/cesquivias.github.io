    Title: Writing a Language in Truffle: Part 1
    Date: 2014-10-09T23:25:17
    Tags: java truffle

Intro
=====

Ever since implementing a [meta circular evaluator](https://mitpress.mit.edu/sicp/full-text/sicp/book/node76.html) in college I've enjoyed implementing little lisp interpreters in languages. It's one of those learning projects I do when learning a new programming language. When I first learned Python and implemented a lisp interpreter I was amazed at how small and easy to read the implementation was. Little did I know that Peter Norvig beat me to [a better python lisp interpreter](http://norvig.com/lispy.html). Nonetheless, it's a fun little exercise to get acquainted with a new programming environment and appeases my curiosity in languages and interpreters.

Of course my toy interpreters are just that: a toy. Writing a lisp interpreter on top of an already slow language like Python will not win any speed competitions. You may get away with writing small Domain Specific Languages (DSLs) as an interpreter, but you can forget about any general programming language. The performance hit makes it untenable unless you write it in some lower level language like C; and who wants to do that? If you want to target higher level virtual machines like the JVM you're left with writing a compiler that takes your code and produces JVM bytecode. Or how about writing a compiler that targets Javascript? Another not-so-fun alternative.

Thankfully, a new solution is here. You can write your interpreter in a VM that is designed to optimize your interpreter with all that wonderful JIT compilation magic. [PyPy](http://pypy.org/) has been around for a few years and has grown into such an environment. It originally started as new Python interpreter written in a smaller, more restricted version of Python appropriately named RPython (R = restricted). You don't get all the syntactic goodness of regular Python but it sure beats writing your interpreter in C. The developers have made herculean efforts to create a VM that can take a high level language like Python and make it fast even when written in a still high level language like RPython. It's so good it's [faster than the standard CPython](http://speed.pypy.org/) implementation in most benchmarks. It's such a promising platform there's an implementation of Ruby called [Topaz](http://topazruby.com/) that is already performing [faster in some benchmarks](http://mail.openjdk.java.net/pipermail/mlvm-dev/2013-February/005214.html) than the much more established implementations like JRuby. Exciting stuff, but now PyPy isn't the only game in town.

Oracle labs has [released its own VM](http://www.oracle.com/technetwork/oracle-labs/program-languages/overview/index-2301583.html) that hopes to achieve the same results as PyPy but that leverages the huge ecosystem of the JVM. Oracle labs has released a custom version of the JVM that contains a new JIT compiler that can speed up interpreters like my little lisp to near-Java speeds. The new JIT compiler is called Graal. To take advantage of Graal's JIT-y goodness you'll have to use the Truffle library to annotation your interpreter and give Graal some hints on invariants and type information. For this integration you get significant speedups in your interpreter without having to resort to writing a bytecode compiler and having the full power of Java at your disposal.

The initial results of Truffle are very exciting. Implementations of Ruby and Javascript in Truffle have performances on the same order of magnitude as the much bigger projects of JRuby and Nashorn, respectively. They are even comparable in speed to more established projects like the Google's V8 Javascript interpreter. The kick: these Truffle implementations were done with fewer people in a shorter period of time. This means you can create your own language on the JVM that takes advantage of all it's existing libraries, native threading, JIT compiler without without having to write your own fullblown optimizer compiler, *and* you get speeds of implementations that took man-years (decades?) to creates. Where do I sign up?

To see if the claims are true I'm going to write another simple Lisp language which I'll unimaginatively call Truffler and see how easy it is. I'll write a simple, non-Truffle interpreter and judge how easy it is to translate to Truffle. I'll take benchmarks to see what kind of speed gains we get by switching over to Truffle. If everything goes well, perhaps this proof of concept will reach speeds comparable to other production-quality lisps like Clojure, Racket or CHICKEN (scheme), but I'm getting ahead of myself.

Truffler Language
=================

I'm going to try to try a tough balance between simplicity and useful features. I don't want a language that's too simple that it's a strawman, but I don't want to spend too much time writing a language I intend to throw away. Truffler will be a simple but fullblown language. I'll stick to basic datatypes and little-to-no syntactic sugar.

For those unfamiliar with lisps, the language will have a syntax as simple as possible and only a couple of datatypes. Lisps can go very far with what would normally be considered an anemic set of types and operations.

Datatypes
---------

  - Numbers (e.g., `1`, `400`)

    Truffler will stick with positive integer literals. You can use the `-` function to create negative numbers if need be. This will be implemented by boxed Long. It's slow, but we're shooting for a barebones language. We also won't deal with integer overflows.

  - Boolean (i.e., `#t`, `#f`)

    Nothing to special here. The tokens `#t` and `#f` will evaluate to the Java static final values `Boolean.TRUE` and `Boolean.FALSE`, respectively.

  - Symbol (e.g., `var`, `some-variable`)

    Symbols are kind of unique to lisps. If you're not familiar with them, they're string-like but are typically used for keys in maps. Think of symbols in Ruby if you're familiar with Ruby but without the `:` in the front. Symbols are show their uniqueness when they're evaluated.

  - Function (e.g., `(lambda (x) (+ x 1))`)

    Functions are written with the `lambda` special form. Functions are first class values that can be stored in a variable, passed to another function or returned in a function.

  - List (e.g., `(1 2 3)`, `(a list of symbols)`)

    List are the only way to make compound structures in Truffler. It's a little limiting but it's plenty for our little language.

    Lists are implemented as singly-linked lists. There are pros and cons with this implementation, but it's the traditional implementation in lisps. We're not blazing any trails here with language design so we'll stick with the tried and true implementation.

Syntax
------

The syntax is simple.

  - Tokens are separated by whitespace or opening/closing parentheses.
  - Numbers are tokens that are digits. Only numbers can begin with a digit.
  - Booleans are the tokens `#t` `#f`. Any other token that begins with `#` is illegal.
  - An open paren `(` starts a new list. A `)` is closes the list. Lists can be nested.
  - Everything else is a symbol.

Here is some syntactically correct Truffler code:

        12344
        -123
        the-above-is-a-symbol-because-Truffler-does-not-have-negative-numbers
        #t
        (a list of symbols)

Evaluation
----------

To keep with our theme of simple, evaluation is as basic as we can make it. The program is evaluated from top to bottom. That means that a variable definition must appear before its use. Everything is an expression. Since Truffler programs are made of Truffler datatypes every datatype has a well-defined evaluation strategy.

  - Number

    Numbers evaluate to themselves. That is, a number returns a number. Shocking, I know.

  - Boolean

    Booleans evaluate to themselves. Same as above.

  - Symbol

    Symbols evaluate by returning looking the current namespace and return the value with the same name as the symbol. When you say `some-variable` the interpreter will see a symbol and try to evaluate it by looking in the current namespace for something stored under `"some-variable"`.

  - List

  Lists are evaluated as function calls. Every element in the list is evaluated before the function is applied to its arguments. The first element must evaluate to a function. The rest of the elements are the arguments. The arguments are mapped to the formal parameter names before the body of the function is called. The last expression in the function body is the return value.

Scoping
-------

Truffler has lexical scope. If you've used any modern dynamic languages like Javascript, Python or Ruby then you know how this works. Basically, it means that functions defined within a wrapping function has access to the outer function's variables--even if the inner function is the return value and is not called until some time in the future.

Truffler doesn't have blocks (the list of statements within `{}` in languages like Java, C, C++). New scopes are only created within new functions.

Special Forms
-------------

The evaluation process for function calls of evaluating all the arguments doesn't work in certain circumstances. In these cases we've defined unique evaluation schemes typically called special forms.

  - `define`

    This creates a new variable within the current scope. `define` has to be a special form because the variable name doesn't exist yet (which is why we're calling `define`) and would throw an error for an undefined variable.

    Example: `(define new-var 42)`

  - `lambda`

    This creates an anonymous function. The second element (the first being the symbol `lambda`) is the list of formal parameters. Lambda will be used with `define` to store a function with a name. `lambda` has to be a special form because we don't want to evaluate the body immediately; only when the function is the first element in a function call. Also, the arguments need to be mapped to the parameter names before the body can be evaluated or else we would get undefined variable errors.

    The name `lambda` is used for historical purposes. We'll just say that it's the traditional name for anonymous functions in lisps and leave it at that.

    Example: `(lambda (x) (* x x))`

  - `if`

    The only control flow structure in Truffle. This works as you'd expect, the second element (the test expression) is evaluated, and if it's true the 3rd element (the "then" expression) is evaluated. If it's false the 4th element is evaluated. `if` has to be a special form because only the then or the else expression must be evaluated. If `if` was a normal function both clauses would be evaluated, and that would be bad.

    Remember, `if` is an expression like everything else. It returns the result of evaluation of either the then or else expression. Lisp ifs are more like the C-style ternary operator: `test() ? then_result() : else_result`.

    Example: `(if (= x 0) (+ x 1) (- x 3))`

  - `quote`

    This returns the argument with evaluation. Since all instances of Symbol and List types in Truffler evaluate to variable lookup and function calls, respectively, we need a way to get a hold of actual Symbol and List objects. `quote` allows us to do that. With lists you can use the `list` builtin function to get a list object, but symbols needs quote because there's no other way to get a Symbol object.

    Keep in mind, quote doesn't evaluate anything in its argument. That means if you pass a list it's elements are not evaluated. A symbol within an list will stay a simple. You're better off using the `list` function unless you really want a list of symbols.

    Because numbers and booleans evaluate to themselves, quoting them doesn't do anything.

With these few special forms we have a fully functional, Turing-complete programming language. We'll have to be creative in combining them to get the functionality we need, but that's part of the power of lisp.

Builtin Functions
-----------------

Although we have a Turing-complete language that isn't much comfort if we can't do something useful with it like interact with the outside world or manipulate our datatypes. Here are the builtin functions that come with Truffler. There's nothing special about these functions other than they already exist when Truffler starts. They are evaluated and called like user-defined functions.

  - Arimetic functions (`+`, `-`, `*`, `/`, `%`)

    We have your basic arithmetic operations. Remember, Truffler only has integers so `/` does integer division and `%` is the modulo function.

  - List functions (`list`, `cons`, `car`, `cdr`)

    Our basic functions for creating, prepending and splicing lists. The names `cons`, `car`, `cdr` probably don't make sense if you're unfamiliar with lisp but they basically mean `prepend`, `first`, `rest` respectively. Again, we're sticking with lisp tradition here. For more background, you can read the Wikpedia article on [cons](http://en.wikipedia.org/wiki/Cons).

  - IO functions (`println`)

    Since we're not planning to do any real work in Truffler, we'll just implement a function to send data to standard out and call it a day.

I may have to add more in the future but this is a good start.


SimpleTruffler
==============

Now with a clear idea of what language we're creating, let's get started! We'll start with an interpreter that _doesn't_ use Truffle as a base comparison. This version will be written the way I would normally write toy langauges, à la Peter Norvig's [lispy](http://norvig.com/lispy.html).


Design
------

The architecture will be a simple pipline from program text to evaluated result.

        Read program text -> evaluate AST

The `Reader` class will encapsulate converting from program text and return the AST. The AST top node will be called with a default environment that includes all builtin functions. When interpreting a file the result of the evaluation will be dropped.

Our main class will look like:

```java
public class SimpleTrufflerMain {
    public static void main(String[] args) throws IOException {
        assert args.length == 1 : "Truffler file required";
        runTruffler(args[0]);
    }

    private static void runTruffler(String filename) throws IOException {
        Environment topEnv = Environment.getBaseEnvironment();

        TrufflerList nodes = Reader.read(new FileInputStream(filename));
        for (Node node : nodes) {
            node.eval(topEnv);
        }
    }
}
```


REPL
----

We can add a small addition at basically no cost. We can take the basic flow described above and wrap it in a `while` loop and create a REPL. If you've ever used languages like Python, Ruby or the Javascript console in web browsers you know a REPL. The REPL takes the text passed to it by the user and follows the flow above but then prints out the result instead of dropping it. We wrap it all in a `while` loop so we always return back to the prompt.

Our updated main class is:

```java
public class SimpleTrufflerMain {
    public static void main(String[] args) throws IOException {
        assert args.length < 2 : "SimpleTruffler only accepts 1 or 0 files";
        if (args.length == 0) {
            startREPL();
        } else {
            runTruffler(args[0]);
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
            TrufflerListNode nodes = Reader.read(
                    new ByteArrayInputStream(data.getBytes()));

            // EVAL
            Object result = ListNode.EMPTY;
            for (Node node : nodes) {
                result = node.eval(topEnv);
            }

            // PRINT
            System.out.println(result);
        }
    }

    private static void runTruffler(String filename) throws IOException {
        Environment topEnv = Environment.getBaseEnvironment();

        TrufflerListNode nodes = Reader.read(new FileInputStream(filename));
        for (Node node : nodes) {
            node.eval(topEnv);
        }
    }
}
```

We're being a little quick and dirty with IOExceptions but the happy path will work fine. Now time to delve into the converting an `InputStream` into an abstract syntax tree.


Reading
-------

We still need to convert out program text into an in-memory tree of expressions. Typically, languages break this up into two stages: lexing (text -> tokens), parsing (tokens -> syntax tree). I'm going to take another page from lisp history and combine lexing and paring into one step: reading. Because we were careful in defining the syntax of Truffler, we can easily hand-write a reader that only needs one character of look ahead to determine what token is being read.

The basic flow of the reader is to read one character then dispatch to specialized readers depending on the character read. Our `readNode` function:

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

I call `unread` to push the read character back onto the stream so it token can be completely read by the called method. I always push it back on to be consistent. Nodes like lists and booleans don't technically need it. If you look at the [list](https://github.com/cesquivias/truffler/blob/master/simple/src/truffler/simple/Reader.java#L76) and [boolean](https://github.com/cesquivias/truffler/blob/master/simple/src/truffler/simple/Reader.java#L114) implementations you can see the only thing we do is assert the first character is what we expected.

If you look closely at our implementation of SimpleTrufflerMain you'll see that we call `Reader.read` and it returns a TrufflerListNode. Our `read` method can read multiple nodes at once and return them all. The `read` method:

```java
public static TrufflerListNode read(InputStream istream) throws IOException {
    return read(new PushbackReader(new InputStreamReader(istream)));
}

private static TrufflerListNode read(PushbackReader pstream)
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

    return TrufflerListNode.list(nodes);
}
```

Within `Reader` we only deal with `PushbackReader` objects. The only public method `read(InputStream istream)` just converts the `InputStream` to a `PushbackReader`.

As you can see the `read` method is straightforward. It accumulates all Node values into a list while the end-of-file hasn't been reached then returns a TrufflerListNode when it's done. It's also responsible for reading and throwing away whitespace so `readNode` doesn't have to worry about it.

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
    return SpecialForm.check(TrufflerListNode.list(list));
}
```

The two interesting bits of this method are (1) we recursively call `readNode` to handle reading all elements in the list. Because start and end parentheses are separators we know that `readList` will get back control once they are reached; (2) We call `SpecialForm.check` to see if the list is not a function but actually one of our four special forms. The code for `SpecialForm.check` is straightforward.

```java
private static final SymbolNode DEFINE = new SymbolNode("define");
private static final SymbolNode LAMBDA = new SymbolNode("lambda");
private static final SymbolNode IF = new SymbolNode("if");
private static final SymbolNode QUOTE = new SymbolNode("quote");

public static Node check(TrufflerListNode l) {
    if (l == TrufflerListNode.EMPTY) {
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

All the nodes the reader returns extend from the abstract class `Node`. All `Node` does is define an abstract method `eval`. `eval` takes an `Environment` argumnet that contains the namespace for all the defined variables. For most datatypes (Number, Boolean, Function) `eval` returns the same object. Actually, Number and Boolean return the boxed values of `java.lang.Long` and `java.lang.Boolean`, respectively, because Java doesn't allow us to subclass `java.lang.Long` and `java.lang.Boolean`. If we could we would just returned `this` in those cases. The difference is subtle and doesn't really matter. For the `Function` type, we do return `this` since we define a custom type.

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


The Environment class is just a simple mapping between strings and objects. Because Truffler is lexically scoped, the Environment's parent is searched if the name cannot be found. If the top parent is reached and the name is never found an exception is thrown.

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

`TrufflerListNode` is evaluated as a function call. This means the that all elements of the list are first evaluated by calling their respective `eval` methods. The first element is then cast to the `Function` class and its `apply` method is called with the rest of the list elements passed as arguments.

```java
public class TrufflerListNode extends Node implements Iterable<Node> {
    // other code...

    @Override
    public Object eval(Environment env) {
        Fn fn = (Fn) this.car.eval(env);

        List<Object> args = new ArrayList<Object>();
        for (Node node : this.cdr) {
            args.add(node.eval(env));
        }
        return fn.apply(args.toArray());
    }
}
```

Builtin functions are subclasses of the `Function` class. User-defined functions are created with the lambda special form. The lambda special form creates an instantiation of an anonymous inner class that subclasses `Function`. These are the only two places that `Function` is implemented. If any other value is the first element of the list a `ClassCastException` is thrown.

The top node's return value of `eval` is returned.


Print
-----

This stage is pretty straightforward. Since `eval` always returns an Object (because we just call `System.out.println` on the returned value. Very simple.


Loop
----

We wrap the read-eval-print portions in a `while (true) {}` loop so we REPL continues indefinitely. If the user enters an EOF (ctrl+D) char we break out of the loop and exit.
