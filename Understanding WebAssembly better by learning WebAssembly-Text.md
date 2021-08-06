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

## What does a "label" actually mean?
All the function calls, parameter and local acces are done by index. Labels
are just compile-time annotations to make code easier to read, write and
understand.

## Are `local.get` and `i32.add` the only instructions?
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

## Globals, linear memory and other low level stuff
