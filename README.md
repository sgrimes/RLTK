# Welcome to the Ruby Language Toolkit

RLTK is a collection of classes and methods designed to help programmers work with languages in an easy to use and straightforward manner.  This toolkit provides the following features:

* Lexer generator
* Parser generator
* AST node baseclass
* Class for representing context free grammars
* [Low Level Virtual Machine](http://llvm.org) (LLVM) bindings for code generation

In addition, RLTK includes several ready-made lexers and parsers and a Turing-complete language called Kazoo for use in your code and as examples for how to use the toolkit.

## Why Use RLTK

Here are some reasons to use RLTK to build your lexers, parsers, and abstract syntax trees:

* **Lexer and Parser Definitions in Ruby** - Many tools require you to write your lexer/parser definitions in their own format, which is then processed and used to generate Ruby code.  RLTK lexers/parsers are written entirely in Ruby and use syntax you are already familiar with.

* **Re-entrant Code** - The lexers and parsers generated by RLTK are fully re-entrant.

* **Multiple Lexers and Parsers** - You can define as many lexers and parses as you want, and instantiate as many of them as you need.

* **Token Positions** - Detailed information about a token's position is available in the parser.

* **Feature Rich Lexing and Parsing** - Often, lexer and parser generators will try and force you to do everything their way.  RLTK gives you more flexibility with features such as states and flags for lexers, and argument arrays for parsers.  What's more, these features actually work (I'm looking at you REX).

* **LALR(1)/GLR Parsing** - RLTK parsers use the LALR(1)/GLR parsing algorithms, which means you get both speed and the ability to handle **any** context-free grammar.

* **Parser Serialization** - RLTK parsers can be serialized and saved after they are generated for faster loading the next time they are required.

* **Error Productions** - RLKT parsers can use error productions to recover from, and report on, errors.

* **Fast Prototyping** - If you need to change your lexer/parser you don't have to re-run the lexer and parser generation tools, simply make the changes and be on your way.

* **Parse Tree Graphs** - RLTK parsers can print parse trees (in the DOT language) of accepted strings.

* **Documentation** - We have it!

* **I Eat My Own Dog Food** - I'm using RLTK for my own projects so if there is a bug I'll most likely be the first one to know.

## Lexers

To create your own lexer using RLTK you simply need to subclass the {RLTK::Lexer} class and define the *rules* that will be used for matching text and generating tokens.  Here we see a simple lexer for a calculator:

	class Calculator < RLTK::Lexer
		rule(/\+/)	{ :PLS }
		rule(/-/)	{ :SUB }
		rule(/\*/)	{ :MUL }
		rule(/\//)	{ :DIV }

		rule(/\(/)	{ :LPAREN }
		rule(/\)/)	{ :RPAREN }

		rule(/[0-9]+/)	{ |t| [:NUM, t.to_i] }

		rule(/\s/)
	end

The {RLTK::Lexer.rule} method's first argument is the regular expression used for matching text.  The block passed to the function is the action that executes when a substring is matched by the rule.  These blocks must return the *type* of the token (which must be in ALL CAPS; see the Parsers section), and optionally a *value*.  In the latter case you must return an array containing the *type* and *value*, which you can see an example of in the Calculator lexer shown above.  The values returned by the proc object are used to build a {RLTK::Token} object that includes the *type* and *value* information, as well as information about the line number the token was found on, the offset from the beginning of the line to the start of the token, and the length of the token's text.  If the *type* value returned by the proc is `nil` the input is discarded and no token is produced.

The {RLTK::Lexer} class provides both {RLTK::Lexer.lex} and {RLTK::Lexer.lex_file}.  The {RLTK::Lexer.lex} method takes a string as its argument and returns an array of tokens, with an *end of stream* token automatically added to the result.  The {RLTK::Lexer.lex_file} method takes the name of a file as input, and lexes the contents of the specified file.

### The Lexing Environment

The proc objects passed to the {RLTK::Lexer.rule} methods are evaluated inside an instance of the {RLTK::Lexer::Environment} class.  This gives you access to methods for manipulating the lexer's state and flags (see bellow).  You can also subclass the environment inside your lexer to provide additional functionality to your rule blocks.  When doing so you need to ensure that you name your new class Environment like in the following example:

	class MyLexer < RLTK::Lexer
		...
		
		class Environment < Environment
			def helper_function
				...
			end
			
			...
		end
	end

### Using States

The lexing environment may be used to keep track of state inside your lexer.  When rules are defined they are defined inside a given state, which is specified by the second parameter to {RLTK::Lexer.rule}.  The default state is cleverly named `:default`.  When the lexer is scanning the input string for matching rules, it only considers the rules for the given state.

The methods used to manipulate state are:

* **RLTK::Lexer::Environment.push_state** - Pushes a new state onto the stack.
* **RLTK::Lexer::Environment.pop_state** - Pops a state off of the stack.
* **RLTK::Lexer::Environment.set_state** - Sets the state at the top of the stack.
* **RLTK::Lexer::Environment.state** - Returns the current state.

States may be used to easily support nested comments.

	class StateLexer < RLTK::Lexer
		rule(/a/)		{ :A }
		rule(/\s/)
		
		rule(/\(\*/)	{ push_state(:comment) }
		
		rule(/\(\*/, :comment)	{ push_state(:comment) }
		rule(/\*\)/, :comment)	{ pop_state }
		rule(/./,    :comment)
	end

By default the lexer will start in the `:default` state.  To change this, you may use the {RLTK::Lexer.start} method.

### Using Flags

The lexing environment also maintains a set of *flags*.  This set is manipulated using the following methods:

* **RLTK::Lexer::Environment.set_flag** - Adds the specified flag to the set of flags.
* **RLTK::Lexer::Environment.unset_flag** - Removes the specified flag from the set of flags.
* **RLTK::Lexer::Environment.clear_flags** - Unsets all flags.

When *rules* are defined they may use a third parameter to specify a list of flags that must be set before the rule is considered when matching substrings.  An example of this usage follows:

	class FlagLexer < RLTK::Lexer
		rule(/a/)		{ set_flag(:a); :A }
		
		rule(/\s/)
		
		rule(/b/, :default, [:a])	{ set_flag(:b); :B }
		rule(/c/, :default, [:a, :b])	{ :C }
	end

### Instantiating Lexers

In addition to using the {RLTK::Lexer.lex} class method you may also instantiate lexer objects.  The only difference then is that the lexing environment used between subsequent calls to {RLTK::Lexer#lex} is the same object, and therefor allows you to keep persistent state.

### First and Longest Match

A RLTK::Lexer may be told to select either the first substring that is found to match a rule or the longest substring to match any rule.  The default behavior is to match the longest substring possible, but you can change this by using the {RLTK::Lexer.match_first} method inside your class definition as follows:

	class MyLexer < RLTK::Lexer
		match_first
		
		...
	end

### Match Data

Because it isn't RLTK's job to tell you how to write lexers and parsers, the MatchData object from a pattern match is available inside the Lexer::Environment object via the `match` accessor.

## Parsers

To create a parser using RLTK simply subclass RLTK::Parser, define the productions of the grammar you wish to parse, and call `finalize`.  During finalization RLTK will build an LALR(1) parsing table, which may contain conflicts that can't be resolved with LALR(1) lookahead sets or precedence/associativity information.  Traditionally, when parser generators such as **YACC** encounter conflicts during parsing table generation they will resolve shift/reduce conflicts in favor of shifts and reduce/reduce conflicts in favor of the production that was defined first.  This means that the generated parsers can't handle ambiguous grammars.

RLTK parsers, on the other hand, can handle *all* context-free grammars by forking the parse stack when shift/reduce or reduce/reduce conflicts are encountered.  This method is called the GLR parsing algorithm and allows the parser to explore multiple different possible derivations, discarding the ones that don't produce valid parse trees.  GLR parsing is more expensive, in both time and space requirements, but these penalties are only payed when a parser for an ambiguous grammar is given an input with multiple parse trees, and as such most parsing should proceed using the faster LALR(1) base algorithm.

### Defining a Grammar

Let us look at the simple prefix calculator included with RLTK:

	class PrefixCalc < RLTK::Parser
		production(:e) do
			clause('NUM') {|n| n}
			
			clause('PLS e e') { |_, e0, e1| e0 + e1 }
			clause('SUB e e') { |_, e0, e1| e0 - e1 }
			clause('MUL e e') { |_, e0, e1| e0 * e1 }
			clause('DIV e e') { |_, e0, e1| e0 / e1 }
		end
		
		finalize
	end

The parser uses the same method for defining productions as the {RLTK::CFG} class.  In fact, the parser forwards the {RLTK::Parser.production} and {RLTK::Parser.clause} method invocations to an internal {RLTK::CFG} object after removing the parser specific information.  To see a detailed description of grammar definitions please read the Context-Free Grammars section bellow.

It is important to note that the proc objects associated with productions should evaluate to the value you wish the left-hand side of the production to take.

The default starting symbol of the grammar is the left-hand side of the first production defined (in this case, _e_).  This can be changed using the {RLTK::Parser.start} function when defining your parser.

**Make sure you call `finalize` at the end of your parser definition, and only call it once.**

### Precedence and Associativity

To help you remove ambiguity from your grammars RLTK lets you assign precedence and associativity information to terminal symbols.  Productions then get assigned precedence and associativity based on either the last terminal symbol on the right-hand side of the production, or an optional parameter to the {RLTK::Parser.production} or {RLTK::Parser.clause} methods.  When an {RLTK::Parser} encounters a shift/reduce error it will attempt to resolve it using the following rules:

1. If there is precedence and associativity information present for all reduce actions involved and for the input token we attempt to resolve the conflict using the following rule.  If not, no resolution is possible and the parser generator moves on.  This conflict will later be reported to the programmer.

2. The precedence of the actions involved in the conflict are compared (a shift action's precedence is based on the input token), and the action with the highest precedence is selected.  If two actions have the same precedence the associativity of the input symbol is used: left associativity means we select the reduce action, right associativity means we select the shift action, and non-associativity means that we have encountered an error.

To assign precedence to terminal symbols you can use the {RLTK::Parser.left}, {RLTK::Parser.right}, and {RLTK::Parser.nonassoc} methods inside your parser class definition.  Later declarations of associativity have higher levels of precedence than earlier declarations of the same associativity.

Let's look at the infix calculator example now:

	class InfixCalc < Parser
		
		left :PLS, :SUB
		right :MUL, :DIV
		
		production(:e) do
			clause('NUM') { |n| n }
			
			clause('LPAREN e RPAREN') { |_, e, _| e }
			
			clause('e PLS e') { |e0, _, e1| e0 + e1 }
			clause('e SUB e') { |e0, _, e1| e0 - e1 }
			clause('e MUL e') { |e0, _, e1| e0 * e1 }
			clause('e DIV e') { |e0, _, e1| e0 / e1 }
		end
		
		finalize
	end

Here we use associativity information to properly deal with the different precedence of the addition, subtraction, multiplication, and division operators.  The PLS and SUB terminals are given left associativity with precedence of 1 (by default all terminals and productions have precedence of zero, which is to say no precedence), and the MUL and DIV terminals are given right associativity with precedence of 1.

### Array Arguments

By default the proc objects associated with productions are passed one argument for each symbol on the right-hand side of the production.  This can lead to long, unwieldy argument lists.  To have your parser pass in an array of the values of the right-hand side symbols as the only argument to procs you may use the {RLTK::Parser.array_args} method.  It must be invoked before any productions are declared, and affects all proc objects passed to `production` and `clause` methods.

### The Parsing Environment

The parsing environment is the context in which the proc objects associated with productions are evaluated, and can be used to provide helper functions and to keep state while parsing.  To define a custom environment simply subclass {RLTK::Parser::Environment} inside your parser definition as follows:

	class MyParser < RLTK::Parser
		...
		
		finalize
		
		class Environment < Environment
			def helper_function
				...
			end
			
			...
		end
	end

(The definition of the Environment class may occur anywhere inside the MyParser class definition.)

### Instantiating Parsers

In addition to using the {RLTK::Parser.parse} class method you may also instantiate parser objects.  The only difference then is that the parsing environment used between subsequent calls to {RLTK::Parser#parse} is the same object, and therefor allows you to keep persistent state.

### Finalization Options

The {RLTK::Parser.finalize} method has several options that you should be aware of:

* **explain** - Value should be `true`, `false`, an `IO` object, or a file name.  Default value is `false`.  If a non `false` (or `nil`) value is specified `finalize` will print an explanation of the parser to $stdout, the provided `IO` object, or the specified file.  This explanation will include all of the productions defined, all of the terminal symbols used in the grammar definition, and the states present in the parsing table along with their items, actions, and conflicts.

* **lookahead** - Either `true` or `false`.  Default value is `true`.  Specifies whether the parser generator should build an LALR(1) or LR(0) parsing table.  The LALR(1) table may have the same actions as the LR(0) table or fewer reduce actions if it is possible to resolve conflicts using lookahead sets.

* **precedence** - Either `true` or `false`.  Default value is `true`.  Specifies whether the parser generator should use precedence and associativity information to solve conflicts.

* **use** - Value should be `false`, the name of a file, or a file object.  If the file exists and hasn't been modified since the parser definition was RLTK will load the parser definition from the file, saving a bunch of time.  If the file doesn't exist or the parser has been modified since it was last used RLTK will save the parser's data structures to this file.

### Parsing Options

The {RLTK::Parser.parse} and {RLTK::Parser#parse} methods also have several options that you should be aware of:

* **accept** - Either `:first` or `:all`.  Default value is `:first`.  This option tells the parser to accept the first successful parse-tree found, or all parse-trees that enter the accept state.  It only affects the behavior of the parser if the defined grammar is ambiguous.

* **env** - This option specifies the environment in which the productions' proc objects are evaluated.  The RLTK::Parser::parse class function will create a new RLTK::Parser::Environment on each call unless one is specified.  RLTK::Parser objects have an internal, per-instance, RLTK::Parser::Environment that is the default value for this option when calling RLTK::Parser.parse

* **parse_tree** - Value should be `true`, `false`, an `IO` object, or a file name.  Default value is `false`.  If a non `false` (or `nil`) value is specified a DOT language description of all accepted parse trees will be printed out to $stdout, the provided `IO` object, or the specified file.

* **verbose** - Value should be `true`, `false`, an `IO` object, or a file name.  Default value is `false`.  If a non `false` (or `nil`) value is specified a detailed description of the actions of the parser are printed to $stdout, the provided `IO` object, or the specified file as it parses the input.

### Parsing Exceptions

Calls to {RLTK::Parser.parse} may raise one of four exceptions:

* **RLTK::BadToken** - This exception is raised when a token is observed in the input stream that wasn't used in the language's definition.
* **RLTK::HandledError** - This exception is raised whenever an error production is encountered.  The input stream is not actually in the langauge, but we were able to handle the encountered errors in a way that makes it appear that it is.
* **RLTK::InternalParserError** - This exception tells you that something REALLY went wrong.  Users should never receive this exception.
* **RLTK::NotInLanguage** - This exception indicates that the input token stream is not in the parser's language.

### Error Productions

**Warning: this is the lest tested feature of RLTK.  If you encounter any problems while using it, please let me know so I can fix any bugs as soon as possible.**

When an RLTK parser encounters a token for which there are no more valid tokens (and it is on the last parse stack / possible parse-tree path) it will enter error handling mode.  In this mode the parser pops states and input off of the parse stack (the parser is a pushdown automaton after all) until it finds a state that has a shift action for the `ERROR` terminal.  A dummy `ERROR` terminal is then placed onto the parse stack and the shift action is taken.  This error token will have the position information of the token that caused the parser to enter error handling mode.

If the input (including the `ERROR` token) can be reduced immediately the associated error handling proc is evaluated and we continue parsing.  If the parser can't immediately reduce it will begin shifting tokens onto the input stack.  This may cause the parser to enter a state in which it again has no valid actions for an input.  When this happens it enters error handling mode again and pops states and input off of the stack until it reaches an error state again.  In this way it searches for the first substring after the error occurred for which it can resume parsing.

The example below for the unit tests shows a very basic usage of error productions:

	class AfterPlsError < StandardError; end
	class AfterSubError < StandardError; end

	class ErrorCalc < RLTK::Parser
		production(:e) do
			clause('NUM') { |n| n }
			
			clause('e PLS e') { |e0, _, e1| e0 + e1 }
			clause('e SUB e') { |e0, _, e1| e0 - e1 }
			clause('e MUL e') { |e0, _, e1| e0 * e1 }
			clause('e DIV e') { |e0, _, e1| e0 / e1 }
			
			clause('e PLS ERROR') { |_, _, _| raise AfterPlsError }
			clause('e SUB ERROR') { |_, _, _| raise AfterSubError }
		end
		
		finalize
	end

## ASTNode

The {RLTK::ASTNode} base class is meant to be a good starting point for implementing your own abstract syntax tree nodes.  By subclassing {RLTK::ASTNode} you automagically get features such as tree comparison, notes, value accessors with type checking, child node accessors and `each` and `map` methods (with type checking), and the ability to retrieve the root of a tree from any member node.

To create your own AST node classes you subclass the {RLTK::ASTNode} class and then use the {RLTK::ASTNode.child} and {RLTK::ASTNode.value} methods.  By declaring the children and values of a node the class will define the appropriate accessors with type checking, know how to pack and unpack a node's children, and know how to handle constructor arguments.

Here we can see the definition of several AST node classes that might be used to implement binary operations for a language:
	
	class Expression < RLTK::ASTNode; end
	
	class Number < Expression
		value :value, Fixnum
	end
	
	class BinOp < Expression
		value :op, String
		
		child :left,  Expression
		child :right, Expression
	end

The assignment functions that are generated for the children and values perform type checking to make sure that the AST is well-formed.  The type of a child must be a subclass of the {RLTK::ASTNode} class, whereas the type of a value must NOT be a subclass of the {RLTK::ASTNode} class.  While child and value objects are stored as instance variables it is unsafe to assign to these variables directly, and it is strongly recommended to always use the accessor functions.

When instantiating a subclass of {RLTK::ASTNode} the arguments to the constructor should be the node's values (in order of definition) followed by the node's children (in order of definition).  Example:

	class Foo < RLTK::ASTNode
		value :a, Fixnum
		child :b, Bar
		value :c, String
		child :d, Bar
	end
	
	Foo.new(1, 'baz', nil, nil)

You may notice that in the above example the children were set to **nil** instead of an instance of the Bar class.  This allows you to specify optional children.

Lastly, the type of a child or value can be defined as an array of objects of a specific type as follows:

	class Foo < RLTK::ASTNode
		value :strings, [String]
	end

## Code Generation

RLTK supports the generation of native code and LLVM IR, as well as JIT compilation and execution, through the {RLTK::CG} module.  This module is built on top of bindings to [LLVM](http://llvm.org) and provides much, though not all, of the functionality of the LLVM libraries.

A very small amount of the functionality of the RLTK::CG module (currently only the {RLTK::CG::Support.load_library} method) requires the [LLVM Extended C Bindings](https://github.com/chriswailes/llvm-ecb) library.  If this library is missing the rest of the module should behave properly, but this functionality will be missing.

### Acknowledgments and Discussion

Before we get started with the details, I would like to thank [Jeremy Voorhis](https://github.com/jvoorhis/).  The bindings present in RLTK are really a fork of the great work that he did on [ruby-llvm](https://github.com/jvoorhis/ruby-llvm).

Why did I fork ruby-llvm, and why might you want to use the RLTK bindings over ruby-llvm?  There are a couple of reasons:

* **Cleaner Codebase** - The RLTK bindings present a cleaner interface to the LLVM library by conforming to more standard Ruby programming practices, providing better abstractions and cleaner inheritance hierarchies, overloading constructors and other methods properly, and performing type checking on objects to better aid in debugging.
* **Documentation** - RLTK's bindings provide slightly better documentation.
* **Completeness** - The RLTK bindings provide several features that are missing from the ruby-llvm project.  These include the ability to initialize LLVM for architectures besides x86 (RLTK supports all architectures supported by LLVM) and the presence of all of LLVM's optimization passes.
* **Ease of Use** - Several features have been added to make generating code easier.
* **Speed** - The RLTK bindings are ever so slightly faster due to avoiding unnecessary FFI calls.

Before you dive into generating code, here are some resources you might want to look over to build up some background knowledge on how LLVM works:

* [Static Single Assignment Form](http://en.wikipedia.org/wiki/Static_single_assignment_form)
* [LLVM Intermediate Representation](http://llvm.org/docs/LangRef.html)

### LLVM

Since RLTK's code generation functionality is built on top of LLVM the first step in generating code is to inform LLVM of the target architecture.  This is accomplished via the {RLTK::CG::LLVM.init} method, which is used like this: `RLTK::CG::LLVM.init(:PPC)`.  The {RLTK::CG::Bindings::ARCHS} constant provides a list of supported architectures.  This call must appear before any other calls to the RLTK::CG module.

If you would like to see what version of LLVM is targeted by your version of RLTK you can either call the {RLTK::CG::LLVM.version} method or looking at the {RLTK::LLVM\_TARGET\_VERSION} constant.

### Modules

Modules are one of the core building blocks of the code generation module.  Functions, constants, and global variables all exist inside a particular module and, if you use the JIT compiler, a module provides the context for your executing code.  New modules can be created using the {RLTK::CG::Module#initialize RLTK::CG::Module.new} method.  While this method is overloaded you, as a library user, will always pass it a string as its first argument.  This allows you to name your modules for easier debugging later.

Once you have created you can serialize the code inside of it into *bitcode* via the {RLTK::CG::Module#write\_bitcode} method.  This allows you to save partially generated code and then use it later.  To load a module from *bitcode* you use the {RLTK::CG::Module.read\_bitcode} method.

### Types

Types are an important part of generating code using LLVM.  Functions, operations, and other constructs use types to make sure that the generated code is sane.  All types in RLTK are subclasses of the {RLTK::CG::Type} class, and have class names that end in "Type".  Types can be grouped into to categories: fundamental and composite.

Fundamental types are those like {RLTK::CG::Int32Type} and {RLTK::CG::FloatType} that don't take any arguments when they are created.  Indeed, these types are represented using a Singleton class, and so the `new` method is disabled.  Instead you can use the `instance` method to get an instantiated type, or simply pass in the class itself whenever you need to reference the type.  In this last case, the method you pass the class to will instantiate the type for you.

Composite types are constructed from other types.  These include the {RLTK::CG::ArrayType}, {RLTK::CG::FunctionType}, and other classes.  These types you must instantiate directly before they can be used, and you may not simply pass the type class as the type argument to functions inside the RLTK::CG module.

For convenience, the native integer type of the host platform is made available via {RLTK::CG::NativeIntType}.

### Values

The {RLTK::CG::Value} class is the common ancestor of many classes inside the RLTK::CG module.  The main way in which you, the library user, will interact with them is when creating constant values.  Here is a list of some of value classes you might use:

* {RLTK::CG::Int1}
* {RLTK::CG::Int8}
* {RLTK::CG::Int16}
* {RLTK::CG::Int32}
* {RLTK::CG::Int64}
* {RLTK::CG::Float}
* {RLTK::CG::Double}
* {RLTK::CG::ConstantArray}
* {RLTK::CG::ConstantStruct}

Again, for convenience, the native integer class of the host platform is made available via {RLTK::CG::NativeInt}.

### Functions

Functions in LLVM are much like C functions; they have a return type, argument types, and a body.  Functions may be created in several ways, though they all require a module in which to place the function.

The first way to create functions is via a module's function collection:

	mod.functions.add('my function', RLTK::CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])

Here we have defined a function named 'my function' in the `mod` module.  It takes two native integers as arguments and returns a native integer.  It is also possible to define the type of a function ahead of time and pass it to this method:

	type = RLTK::CG::FunctionType.new(RLTK::CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])
	mod.functions.add('my function', type)

Functions may also be created directly via the {RLTK::CG::Function#initialize RLTK::CG::Function.new} method, though a reference to a module is still necessary:

	mod = Module.new('my module')
	fun = Function.new(mod, 'my function', RLTK::CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])

or
	
	mod  = Module.new('my module')
	type = RLTK::CG::FunctionType.new(RLTK::CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])
	fun  = Function.new(mod, 'my function', type)

Lastly, whenever you use one of these methods to create a function you may give it a block to be executed inside the context of the function object.  This allows for easier building of functions:

	mod.functions.add('my function', RLTK::CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType]) do
		bb = blocks.append('entry)'
		...
	end

### Basic Blocks

Once a function has been added to a module you will need to add {RLTK::CG::BasicBlock BasicBlocks} to the function.  This can be done easily:

	bb = fun.blocks.append('entry')

We now have a basic block that we can use to add instructions to our function and get it to actually do something.  You can also instantiate basic blocks directly:

	bb = RLTK::CG::BasicBlock.new(fun, 'entry')

### The Builder

Now that you have a basic block you need to add instructions to it.  This is accomplished using a {RLTK::CG::Builder builder}, either directly or indirectly.

To add instructions using a builder directly (this is most similar to how it is done using C/C++) you create the builder, position it where you want to add instructions, and then build them:

	fun = mod.functions.add('add', RLTK:CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])
	bb  = fun.blocks.append('entry')
	
	builder = RLTK::CG::Builder.new
	
	builder.position_at_end(bb)
	
	# Generate an add instruction.
	inst0 = builder.add(fun.params[0], fun.params[1])
	
	# Generate a return instruction.
	builder.ret(inst0)

You can get rid of some of those references to the builder by using the {RLTK::CG::Builder#build} method:

	fun = mod.functions.add('add', RLTK:CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])
	bb  = fun.blocks.append('entry')
	
	builder = RLTK::CG::Builder.new
	
	builder.build(fun) do |fun|
		ret add(fun.params[0], fun.params[1])
	end

To get rid of more code:

	fun = mod.functions.add('add', RLTK:CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])
	bb  = fun.blocks.append('entry')
	
	RLTK::CG::Builder.new do
		ret add(fun.params[0], fun.params[1])
	end

Or:

	fun = mod.functions.add('add', RLTK:CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])
	fun.blocks.append('entry') do |fun|
		ret add(fun.params[0], fun.params[1])
	end

Or even:

	mod.functions.add('add', RLTK:CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType]) do
		blocks.append('entry', nil, nil, self) do |fun|
			ret add(fun.params[0], fun.params[1])
		end
	end

In the last two examples a new builder object is created for the block.  It is possible to specify the builder to be used:

	builder = RLTK::CG::Builder.new
	
	fun = mod.functions.add('add', RLTK:CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType])
	fun.blocks.append('entry', nil, builder) do |fun|
		ret add(fun.params[0], fun.params[1])
	end

For an example of where this is useful, see the Kazoo tutorial.

### Execution Engines

Once you have generated your code you may want to run it.  RLTK provides bindings to both the LLVM interpreter and JIT compiler to help you do just that.  Creating a JIT compiler is pretty simple.

	mod = RLTK::CG::Module.new('my module')
	jit = RLTK::CG::JITCompiler(mod)
	
	mod.functions.add('add', RLTK:CG::NativeIntType, [RLTK::CG::NativeIntType, RLTK::CG::NativeIntType]) do
		blocks.append('entry', nil, nil, self) do |fun|
			ret add(fun.params[0], fun.params[1])
		end
	end

Now you can run your 'add' function like this:

	jit.run(fun, 1, 2)

The result will be a {RLTK::CG::GenericValue} object, and you will want to use its {RLTK::CG::GenericValue#to\_i #to\_i} and {RLTK::CG::GenericValue#to\_f #to\_f} methods to get the Ruby value result.

## Context-Free Grammars

The {RLTK::CFG} class provides an abstraction for context-free grammars.  For the purpose of this class terminal symbols appear in **ALL CAPS**, and non-terminal symbols appear in **all lowercase**.  Once a grammar is defined the {RLTK::CFG#first_set} and {RLTK::CFG#follow_set} methods can be used to find *first* and *follow* sets.

### Defining Grammars

A grammar is defined by first instantiating the {RLTK::CFG} class.  The {RLTK::CFG#production} and {RLTK::CFG#clause} methods may then be used to define the productions of the grammar.  The `production` method can take a Symbol denoting the left-hand side of the production and a string describing the right-hand side of the production, or the left-hand side symbol and a block.  In the first usage a single production is created.  In the second usage the block may contain repeated calls to the `clause` method, each call producing a new production with the same left-hand side but different right-hand sides.  {RLTK::CFG#clause} may not be called outside of {RLTK::CFG#production}.  Bellow we see a grammar definition that uses both methods:

	grammar = RLTK::CFG.new
	
	grammar.production(:s) do
		clause('A G D')
		clause('A a C')
		clause('B a D')
		clause('B G C')
	end
	
	grammar.production(:a, 'b')
	grammar.production(:b, 'G')

### Extended Backus–Naur Form

The RLTK::CFG class understands grammars written in the extended Backus–Naur form.  This allows you to use the \*, \+, and ? operators in your grammar definitions.  When each of these operators are encountered additional productions are generated.  For example, if the right-hand side of a production contained 'NUM*' a production of the form 'num_star -> | NUM num_star' is added to the grammar.  As such, your grammar should not contain productions with similar left-hand sides (e.g. foo_star, bar_question, or baz_plus).

As these additional productions are added internally to the grammar a callback functionality is provided to let you know when such an event occurs.  The callback proc object can either be specified when the CFG object is created, or by using the {RLTK::CFG#callback} method.  The callback will receive three arguments: the production generated, the operator that triggered the generation, and a symbol (:first or :second) specifying which clause of the production this callback is for.

### Helper Functions

Once a grammar has been defined you can use the following functions to obtain information about it:

* **RLTK::CFG.first_set** - Returns the *first set* for the provided symbol or sentence.
* **RLTK::CFG.follow_set** - Returns the *follow set* for the provided symbol.
* **RLTK::CFG.nonterms** - Returns a list of the non-terminal symbols used in the grammar's definition.
* **RLTK::CFG.productions** - Provides either a hash or array of the grammar's productions.
* **RLTK::CFG.symbols** - Returns a list of all symbols used in the grammar's definition.
* **RLTK::CFG.terms** - Returns a list of the terminal symbols used in the grammar's definition.

## Tutorial

What follows is an in-depth example of how to use the Ruby Language Toolkit.  This tutorial will show you how to use RLTK to build a lexer, parser, AST nodes, and compiler to create a toy language called Kazoo. The tutorial is based on the LLVM [Kaleidoscope](http://llvm.org/docs/tutorial/) tutorial, but has been modified to:

* a) be done in Ruby
* 2) use a lexer and parser generator and
* III) use a language that I call Kazoo, which is really just a cleaned up and simplified version of the Kaleidoscope language used in the LLVM tutorial (as opposed to the [Kaleidoscope language](http://en.wikipedia.org/wiki/Kaleidoscope_%28programming_language%29) from the 90′s).

The Kazoo toy language is a procedural language that allows you to define functions, use conditionals, and perform basic mathematical operations.  Over the course of the tutorial we’ll extend Kazoo to support the if/then/else construct, for loops, JIT compilation, and a simple command line interface to the JIT.

Because we want to keep things simple the only datatype in Kazoo is a 64-bit floating point type (a C double or a Ruby float).  As such, all values are implicitly double precision and the language doesn’t require type declarations.  This gives the language a very nice and simple syntax.  For example, the following example computes Fibonacci numbers:

	def fib(x)
		if x < 3 then
			1
		else
			fib(x-1) + fib(x-2)

The tutorial is organized as follows:

  * [Chapter 1: The Lexer](file.Chapter1.html)
  * [Chapter 2: The AST Nodes](file.Chapter2.html)
  * [Chapter 3: The Parser](file.Chapter3.html)
  * [Chapter 4: AST Translation](file.Chapter4.html)
  * [Chapter 5: JIT Compilation](file.Chapter5.html)
  * [Chapter 6: Adding Control Flow](file.Chapter6.html)
  * [Chapter 7: Playtime](file.Chapter7.html)
  * [Chapter 8: Mutable Variables](file.Chapter8.html)

Before starting this tutorial you should know about regular expressions, the basic ideas behind lexing and parsing, and be able to read context-free grammar (CFG) definitions.  By the end of this tutorial we will have written 372 lines of source code and have a JIT compiler for a Turing complete language.

## Provided Lexers and Parsers

The following lexer and parser classes are included as part of RLTK:

* {RLTK::Lexers::Calculator}
* {RLTK::Lexers::EBNF}
* {RLTK::Parsers::PrefixCalc}
* {RLTK::Parsers::InfixCalc}
* {RLTK::Parsers::PostfixCalc}

## Contributing

If you are interested in contributing to RLTK you can:

* Help provide unit tests.  Not all of RLTK is tested as well as it could be.  Specifically, more tests for the RLTK::CFG and RLTK::Parser classes would be appreciated.
* Write lexers or parsers that you think others might want to use.  Possibilities include HTML, JSON/YAML, Javascript, and Ruby.
* Write a class for dealing with regular languages.
* Extend the RLTK::CFG class with additional functionality.
* Let me know if you found any part of this documentation unclear or incomplete.