# Understanding WebAssembly better by learning WebAssembly Text

WebAssembly is a true revolution in tech, not just in the web, but thanks to WASI and friends, it is becoming available everywhere.
One of the best things WebAasembly offers is being a compilation target 
instead of just another programming language. This has the potential to help a lot of non-JS developers 
get involved with web development. WebAssembly also has its text version 
called... You got it, WebAssembly Text, or WAT for short! (official docs at Mozilla [here](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format)).
It can be compiled to the binary format using [wabt](https://github.com/WebAssembly/wabt).

**WARNING**: This article assumes that you already know how to load and use WebAssembly binaries.

## Understanding the Syntax
WAT offers two ways of writing code:
The traditional assembly style
```wat
	local.get 0
	local.get 1
	i32.add
```
and a more LISPy way
```wat
	(i32.add
		(local.get 0)
		(local.get 1))
```
The assembler will spit out the same result from both of them, but the first
example shows in a more clear way how the instructions are placed in the
binary. We will be using the latter, because our goal is to understand how WebAssembly works.
The most basic, valid, (albeit useless) WAT file has the contents
below:
```wat
(module)
```
## Hello, world! Well, sort of.
WebAssembly Text files always start with the module definition and everything else is put between the `module` word and the last parenthesis. Let's see how we can write a simple "Hello, world!" program in WAT.
```wat
(module
	(func $hello_world (param $0 i32) (param $1 i32) (result i32)
		local.get $0
		local.get $1
		i32.add)
	(export "helloWorld" (func $hello_world)))
```
Okay, you might be saying "What the hell is this? I thought this is a 'Hello, world!' example!".
Well, the point is that WASM wasn't created to print strings and interact with
APIs, its purpose is to help JavaScript handle heavy computations by providing
an interface to fast, low level instructions.
### But what does the code do?
 - `func` declares a function
 - `$hello_world` is a compile time name/label we give the function (we'll see that later)
 - `(param $0 i32)` and `(param $1 i32)` tell that this function accepts two parameters, the first one labeled $0 (notice the `$`) with a type `i32`,
 	the second one labeled $1 with a type `i32`.
 - `(result i32)` says that the function return type is an `i32`.
 - `local.get $0` pushes the value of the parameter labeled as $0 into the stack.
 - `local.get $1` does the same as above, but instead it pushes the value of $1.
 - `i32.add` pops two values of type `i32` from the stack and pushes the result of their addition back into the stack.
 - `(export "helloWorld" (func $hello_world))` exports the function labeled as `$hello_world` to the host with the name "helloWorld".

### Where is the `return` statement
WebAssembly has a `return` instruction, but it is mostly used when you need
to immediatly return and stop executing the function any further. Otherwise,
there will be an implicit return, where the last value is poped from the stack
and returned.

### What does a "label" actually mean?
All the function calls, parameter and local acces are done by index. Labels
are just compile-time annotations to make code easier to read, write and
understand.

### Are `local.get` and `i32.add` the only instructions?
Of course not. The WebAssembly instruction set is really big. Just the MVP
has around 120 instructions. Most of them start with the type, which can only
be `i32`, `i64`, `f32`, `f64`, followed by a dot and the name of the
operation. There are also instructions for other purpouses, like control flow,
branching, jumping and looping. A list of all the instructions and their
explanation can be found [here](https://github.com/sunfishcode/wasm-reference-manual/blob/master/WebAssembly.md#instructions).

## Writing something useful
The example below shows how you can write a function that calculates the
distance between two points using Pythagoras formula.
```wat
(module
	(func $distance (param $x1 i32) (param $y1 i32)
			(param $x2 i32) (param $y2 i32) (result f64)

		local.get $x1
		local.get $x2
		i32.sub ;; calculate the X axis distance (a)
		call $square ;; (a ^ 2)

		local.get $y1
		local.get $y2
		i32.sub ;; calculate the Y axis distance (b)
		call $square ;; (b ^ 2)

		i32.add ;; (c^2 = a^2 + b^2)
		f64.convert_s/i32 ;; convert to f64 so we can square root
		f64.sqrt)
	(export "distance" (func $distance))
	(func $square (param $i32) (result i32)
		local.get 0
		local.get 0
		i32.mul))
```
### Breaking down the code (again)
 - `;;` is a the start of a single line comment
 - `i32.sub` pops two numbers of type `i32` from the stack and pushes back the
 	result of their subtraction
 - `call $square` calls a function
 - `f64.convert_s/i32` pops an `i32` from the stack and pushes back its value
   as a signed number converted to an `f64` (if it would have been
   `f64.convert_u/i32` it would convert it from its unsigned form).
   If you don't understand the difference between signed and unsigned numbers
   (in memory), I sugges you read [this](https://dev.to/aidiri/signed-vs-unsigned-bit-integers-what-does-it-mean-and-what-s-the-difference-41a3)
 - `f64.sqrt` pops a number of type `f64` from the stack and pushes the
   number's square root.
 - `i32.mul` pops two numbers of type `i32` from the stack and pushes back the
   result of their multiplication.
   (we multiply a number with himself to get its power of 2 in the `$square` function).

## Decision making
Although WebAssembly is a low level bytecode format, it supports higher level
concepts like if statements and loops. The code below shows a function
that recives two `i32` as parameters and returns the largest of them.
```wat
(module
	(func $largest (param $0 i32) (param $1 i32) (result i32)
		local.get $0 ;; pushing $0's value into the stack
		local.get $1 ;; pushing $1's value into the stack
		i32.gt_s ;; comparing if $0 is greater than $1
		if (result i32)
			local.get $0
		else
			local.get $1
		end)
	(export "largest" (func $largest)))
```
The 3 first instructions are enough simple to be explained with comments,
otherwise:
 - `local.get $0` and `local.get $1` push the values of the parameters labeled
   `$0` and `$1`
 - `i32.gt_s` pops two values of type `i32` from the stack and pushes back
   the result of their comparison (value of `param $0` is greater than the
   valule of `param $1`). `1` if true, `0` if false. It has the `_s` suffix
   because it compares the numbers as signed ones (i.e they can be negative).

### To return `$0` or to return `$1`
When an `if` instruction occurs, the last item in the stack is poped. It needs
an `i32`. If it is different from `0` the instructions inside the `if..else/end` block are executed, otherwise the `else..end` is executed. If there isn't
an `else`, simply nothing is executed.
You might notice that the if "statement" has a result. That is done because
when an `if` instruction is executed, the stack is marked/splited at the
current stack top and a new isolated empty stack context is created, which we
will call the child stack. When the `if` block is executed, the last item in
the child stack is poped, that stack is unwinded and destroyed and the poped
item is pushed to the parent stack. Thats how functions worok, but when
a function is executed, the parameters are first consumed from the parent
stack.

### Select
For simple decisions `if` might be a bit overkill. There is also a `select`
instruction which works like a ternary operator. If we would use `select`

instead of `if`, we would do it like this.
```wat
local.get $0
local.get $1 ;; push value of $0 and $1 into stack for selection
local.get $0
local.get $1 ;; push value of $0 and $1 into stack for comparing
i32.gt_s ;; doing the comparison between the two last `i32` items on the stack
select
```
The `select` instruction pops 3 values from the stack. It decides which of
the two first values to push back according to the third one. (if the third
one is not `0`, push the first one, otherwise the second one.

## Looping and branching
WebAssembly supports loops but not the kind of loops you might be thinking
about. Take this code for example:
```wat
(module
	(func $fac (param $0 i32) (result i32) (local $acc i32)
		local.get $0
		local.set $acc
		block $outer
			loop $loop
				local.get $acc
				i32.const 1
				i32.le_u
				br_if $outer
				local.get $acc
				local.get $acc
				i32.const 1
				i32.sub
				i32.add
				local.set $acc
				br $loop
			end
		end
		local.get $acc)
	(export "factorial" (func $fac)))
```
This code shows how to write a function that finds the factorial of a number
using a loop. It could have been easier to use recursion, but the point here
is to understand loops and branching.

### Looping and breaking down code...
Most of the things in this code have been explained. The only new things
here are `(local $acc i32)`, `block`, `loop`, `br_if $outer` and `$br $loop`.
 - `(local $acc i32)` is like what thry call a local variable in higher level
   languages. It is accessed the same way as the parameters, and the first
   local has the smallest index that is not a parameter.
 - `block $outer` works kind of like `if`, in fact, `if` and `loop` are like
   specialized `block` instruction. The difference is that `block` does not do
   anything special, it just splits the stack and is usually used for general
   purpouse control flow.
   in code so you can branch out.
 - `loop $loop` behaves like a `block`, but if you do a branch on a `loop`,
   you don't go at the `end`, you go at the top (branches behave like break
   on `block`s and like continue on `loop`s)
 - `br_if $outer` is a conditional branch/jump instruction. If you know
   C/C++ you know about jumping using `goto`. But using `goto` you can jump
   every way, everywhere from every part of the code. WebAssembly is more
   restrictive. You can branch only out. The labels are there to make 
   the code easier to read. Without them, it would have been `br_if 1`.
   As most of the things in WebAssembly, branching is achieved by indexes.
   Indexing starts from the innermost to the outermost `block`. Which means
   that the `loop` has an index `0`.
 - `br $loop` is an unconditional branch. It operates on a `loop`, so
   it will go at the top wherw the `loop` instruction occurs.

<!-- ## Globals, linear memory and other low level stuff
WebAssembly offers another way to store data except the stack, the linear
memory. It can be seen as a resizable JavaScript TypeArray. Its main
purpouse is to store complex and/or continous data. -->
