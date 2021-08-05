# Understanding WebAssembly better by learning WebAssembly-Tezt

WebAssembly is a true revolution in tech, not just in the web, but thanks to WASI and friends, it is becoming available everywhere.
One of the best things WebAasembly offers is being a compilation target 
instead of just another programming language. This made a lot of developers 
get involved with web development. But WebAssembly also has its text version 
called...guess what? WebAssembly-Text, or WAT for short (official docs at Mozilla [here](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format). It can be compiled to the binary format using [wabt](https://github.com/WebAssembly/wabt).

**WARNING**: This article assumes that you already know how to load and use WebAssembly binaries.

## Understanding syntax.
WAT has  LISP syntax and offers two ways of writing code:
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
The compiler will spit out the same result from both of them, but the first
example shows in a more clear way how the instructions are placed in the
binary. We will be using the first one, because our goal is to understand how WebAssembly works.
The most basic, useless but also valid WAT file has at least the contents
below:
```wat
(module)
```
## Hello, world! Well, sort of.
It always starts with the module definition and everything else is put between the `module` word and the last parenthesy. Lets see how we can write a simple "Hello, world!" program in WAT.
```wat
(module
	(func $hello_world (param $0 i32) (param $1 i32) (result i32)
		local.get $0
		local.get $1
		i32.add)
	(export "helloWorld" (func $hello_world)))
```
Okay, you might be saying "What the hell is this? I thought it would be a hello world example".
Well, the point is that WASM wasn't created to print string and stuff,
the reason of its creation was to help JavaScript handle heavy computations
by providing an interface to really low level stuff. A hello world example is
to show a silly example that something works. Thats what this one over here does.
Now, what does this code do ?
 - `func` declares a function
 - `$hello_world` is a compile time name/label we give the function (we'll see that later)
 - `(param $0 i32)` and `(param $1 i32)` tell that this function accepts two parameters, the first one labeled $0 (notice the `$`) with a type `i32`,
 	the second one labeled $1 with a type `i32`.
 - `(result i32)` says that the function return type is an `i32`.
 - `local.get $0` pushes the value of the parameter labeled as $0 into the stack.
 - `local.get $1` does the same as above, but instead it pushes the value of $1.
 - `i32.add` pops two values of type `i32` from the stack and pushes the result of their addition into the stack.
 - `(export "helloWorld" (func $hello_world))` exports the function labeled as `$hello_world` with the name "helloWorld".
