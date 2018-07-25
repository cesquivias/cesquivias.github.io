    Title: Add a Language to GraalVM 1.0.0
    Date: 2018-07-31T01:08:04
    Tags: DRAFT, truffle, graal, mumbler

I want to write more, but life is busy. The announcement of GraalVM RC-1.0.0 renewed my enthusiasm to dive back in and see what's changed. Although most of the API has remained the same, the API to register and retrieve an interpreter has changed dramatically.

This post covers the changes in how to register your interperter and how to get an instance of another interpreter.

<!-- more -->

Truffle Changes: Playing well with others
-----------------------------------------

Previously, to start a Truffle interpreter you would just grab the language's custom parser, parse the source code and then feed it into its custom run method. Now, all languages share the same API so any users of your language don't have to stumble around trying to figure out how to get started.

The release of 1.0.0 also saw a big change in how you registered your language with the system. This makes sense since any users of your language will access it via a system-provided API. How will the system know of your language if you never tell it? Along with registering your language, there are now common hooks for converting source code of a program into a Truffle tree.


Example: A Brainfuck Interpreter
--------------------------------

My original intent was to upgrade my Truffle-based [Mumbler](https://github.com/cesquivias/mumbler) programming language to work with the RC-1.0.0 version of Graal, but the changes to the API were so large it was hard to get my head around them. Instead, I decided to write another interpreter so I can start could focus only on the registration API changes.

I wrote a [Brainfuck (BF) interpreter](https://github.com/cesquivias/bf-graal) since it's a simple language but is fully functional. There are also multiple implementations out there against which I can test mine to make sure it works. With a simple language, I can accomplish my task of learning the new Truffle APIs relatively easily while not getting bogged down in details.

I'll walk through my code related to using and registering my BF interpreter. If you're interested in learning how to write the AST of the interpreter (parsing, tree building & interpreting), you can check out [my previous blog posts where I write Mumbler](http://cesquivias.github.io/tags/truffle.html). All the concepts and APIs are essentially the same. I'll just focus on what's changed in this post.

Getting an Instance of the Interpreter
--------------------------------------

In my previous interpreter, to run a program you'd have to know my interpreter's way of parsing the source code and kicking off the interpretation of the Truffle tree the parser returned. Since I didn't optimize for the usability, the whole process was kind of tedious. In theory, other people would like to use my interpreter so I should probably make it easy to take source code and interpret it. On top of that, one of Graal's selling points is you can use multiple interpreters within one program to complete your task. It would be annoying to have to learn each interpreter's special incantation to spin up its interpreter.

Graal has solved this by creating an API to interface with all interpreters.

The two main compontents of the API are the [`Source`](http://www.graalvm.org/sdk/javadoc/org/graalvm/polyglot/Source.html) class and the [`Context`](http://www.graalvm.org/sdk/javadoc/org/graalvm/polyglot/Context.html) class.

### Creating a `Source`

As the name suggests, `Source` represents the source code of your program as well as access to metadata like line numbers or column offsets. For more involved languages, these methods aren't that useful since you'll probably use the the methods provided by your parser generator (e.g., ANTLR), but they're nice for small languages.

If you know the special `String` ID the interpreter uses, you can create a `Source` object directly. For my interpreter, my ID is simply `"bf"` so you can create a `Source` object like so:

```java
Source src = Source.newBuilder("bf", new File("/path/to/script.bf")).build();
```

That's pretty simple, though I'm not a big fan of the special ID `String`. Although unlikely, using IDs seems prone to collisions. I don't think the [reverse domain name convention](https://en.wikipedia.org/wiki/Reverse_domain_name_notation) is customary more language IDs. I would've preferred using the interpreter's `TruffleLanguage` subclass, but it's a pretty minor deal.

There's a more user-friendly way to fetch the proper `Source` object: have Graal do it for you. Graal can determine the language based on the filename extension. We'll have to register the extension, but for now assume we have.

```java
File script = new File("/path/to/script.bf");
Source src = Source.newBuilder(Source.findLanguage(script), script).build();
```

Calling `Source.findLanguage(File file)` avoids having to know the special String ID for the language. This way, you can also avoid custom code to run different interpreters and just have a generic `eval` method if you wanted.

### Creating a `Context`

The `Context` encapsulates the environment and state of the interpreter. There's nothing preventing you from instantiating multiple `Context` objects and running multiple interpreters. Perhaps the language isn't thread safe, so you run separate interpreters on separate threads.

Creating the `Context` is very similar to creating a `Source`

```java
Context context = Context.newBuilder(Source.findLanguage(script))
    .in(System.in)
    .out(System.out)
    .build();
```

Again we use `Source.findLanguage(File file)` to identify the language. We also define the proper `InputStream` and `PrintStream` objects that will be our interpreter's standard in and standard out, respectively.

### Evaluating your Script

After getting our `Source` and `Context` objects we're ready to evaluate. There's not much to it:

```java
context.eval(src);
```

That's it. Create a `Source` object from your source code, create a `Context` object for your langauge, then call `eval` on the `Context` passing in the `Source` object. Nice and simple.

Registering my BF interpreter
-----------------------------

We went over how to use an interpreter that has been registered, but that just begs the question: how do I register my interpreter? This process is a little more involved, but thankfully it's nicely encapsulated in two main classes.

### Defining a Language Class

Truffle makes it easy to register your language with the JVM. You basically define a class that extends [`TruffleLanguage`](https://www.graalvm.org/truffle/javadoc/com/oracle/truffle/api/TruffleLanguage.html) and annotate the class with [`TruffleLanguage.Registration`](https://www.graalvm.org/truffle/javadoc/index.html?com/oracle/truffle/api/TruffleLanguage.Registration.html).

Here's my BF interpreter's entire [`BFLanguage`](https://github.com/cesquivias/bf-graal/blob/master/lang/src/main/java/cesquivias/bf/BFLanguage.java) class, and as I said, it extends `TruffleLanguage` and is annotated with `TruffleLanguage.Registration`.


```java
@TruffleLanguage.Registration(id = ID, name = "BF", version = "1.0-SNAPSHOT",
        mimeType = BFLanguage.MIME_TYPE)
public class BFLanguage extends TruffleLanguage<BFContext> {
    public static final String ID = "bf";
    public static final String MIME_TYPE = "application/x-bf";

    @Override
    protected BFContext createContext(Env env) {
        return new BFContext(this, env);
    }

    @Override
    protected boolean isObjectOfLanguage(Object object) {
        return false;
    }

    @Override
    protected CallTarget parse(ParsingRequest request) throws Exception {
        RootNode rootNode = BFParser.parse(this, request.getSource());
        return Truffle.getRuntime().createCallTarget(rootNode);
    }
}
```

The annotation provides all the information needed to retrieve the language. You define the ID of the language (in my case it's simply `"bf"`), and you give it some metadata like a more readable name and a version. You also need to provide a mime type. Users of the language can even by the mime type. File extension registration isn't done here. We'll do that later, and it will utilize the mime type you set so don't ignore it.

I implement the minimum number of methods: `createContext`, `isObjectOfLanguage` and `parse`. You saw in the previous section that the interpreter runs inside a Context. The system creates that context by calling the `createContext`. Hooray for obvious interfaces.

`isObjectOfLanguage` is used to ask the interpreter if a given object is something the interpreter can handle. For most languages, this method should return true for common data types like Strings, but since my interpreter doesn't use any external data types and doesn't interact with the outside world other than via standard in/out, I have it always return false.

Last, but not least, is `parse`. This is the very important bridge between source code and a Truffle AST that can be interpreted. You'll typically delegate all parsing to some other class as I did. Since this post isn't covering the creation of the AST I'll just move one.

That's it; you can just create a simple `TruffleLanguage` subclass like I have here and you'll have a language that other people can find. Obviously, my example is very minimal and any non-trivial language will implement more methods, but this is the minimal, working class you need and thankfully the code is pretty straightforward.

### Associating your Language with a File Extension

We're at the final piece: registering your interpreter with a file extension. This seems like the most intuitive way--to me--to access the interpreter.  Let's tell the JVM files that end in `.bf` can be run by my interpreter.

To register your language with a file extension, you need to have two files: a subclass of [`FileTypeDectector`](https://docs.oracle.com/javase/8/docs/api/java/nio/file/spi/FileTypeDetector.html) and a special file that goes in your published JAR's META-INF directory.

Our FileTypeDectector is very simple.

```java
public class BFFileDetector extends FileTypeDetector {
    @Override
    public String probeContentType(Path path) {
        if (path.getFileName().toString().endsWith(".bf")) {
            return BFLanguage.MIME_TYPE;
        }
        return null;
    }
}
```

We have to implement one method `probeContentType` and see if the given filename ends with `.bf`. If so, we associate that file with our language's mime type. If not, we just return null and the file won't be associated with our language.

You can have your language associated with any number of extensions (or filename patterns if you'd like). For example, [Racket](http://www.racket-lang.org) files typically end in `.rkt` but also support `.ss`, `.scm` and `.sch` for historical reasons. You could have your Racket interpter associated with all these languages.

Now that we have our `FileTypeDetector` subclass, we need to register it in our published JAR. Inside the JAR, the file would live under `META-INF/services/java.nio.file.spi.FileTypeDetector`. I used Maven to build my BF interpter, and in Maven if you put a file under `$PROJECT_DIR/src/main/resources` it will get included in your JAR.

The contents of the file are simple the fully qualified class name of your `FileTypeDetector` subclass. Therefore, [my file](https://github.com/cesquivias/bf-graal/blob/master/lang/src/main/resources/META-INF/services/java.nio.file.spi.FileTypeDetector) contains

```
cesquivias.bf.BFFileDetector
```

That's it
---------

The Graal team has done a good job of creating an easy way of starting any interpreter. Even on the implementation side, there isn't much to registering or even associating a file extension.

I wish I had more time to play with and write about Graal/Truffle, but that's probably not going to happen. But if I find some time to hack on [Mumbler](https://github.com/cesquivias/mumbler) and discover a handy trick I'll try to write it up here.
