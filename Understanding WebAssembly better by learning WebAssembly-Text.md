# Understanding WebAssembly better by learning WebAssembly Text

WebAssembly is a true revolution in tech, not just in the web, but thanks to WASI and friends, it is becoming available everywhere.
One of the best things WebAssembly offers is being a compilation target 
instead of just another programming language. This has the potential to help a lot of non-JS developers 
get involved with web development. WebAssembly also has its text version 
called... You got it, WebAssembly Text, or WAT for short! (MDN docs [here](https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format)).
It can be compiled to the binary format using [WABT](https://github.com/WebAssembly/wabt).

## Prerequisites (to follow along)
- You know how to use WABT to assemble WAT.
- You know how to run WebAssembly binaries.

## Understanding the Syntax
WAT offers two ways of writing code:
The traditional assembly style
```wat
	local.get 0
	local.get 1
	i32.add
```
and a more LISPy way (called S-Expression format)
```wat
	(i32.add
		(local.get 0)
		(local.get 1))
```
The assembler will spit out the same result from both of them, but the former 
shows in a more clear way how the instructions are placed in the
binary. We will be using that style in this article.
The most basic, valid, (albeit useless) WAT file has the contents below:
```wat
(module)
```

## How WebAssembly's stack works
A stack is nothing more than just a LIFO (last in, first out) linear data
structure. Imagine it as an array where you can only
`push` and `pop` and can't access the items in any other way.
WebAsembly's stack is no difference, but it has a few features which make it
cooler and safer.
One of which is stack splitting/framing. The name may look scary, but it is
simpler than it looks. It just puts a mark at the place where the item at the 
top of the stack is on the moment it is split. The mark is the bottom of the new frame 
and it simply says "Hey, this is a stack of its own that this **block** of
code here operates on. Only this block of code has access to it and nothing
else. The new stack's lifetime is limited to the time that this block of code requires
to be fully executed".
We are going to call the stack that is split the parent frame and the new
stack that results from the split the child frame.
A **block** is the part between a `block`/`if`/`loop` instruction
and an `end` instruction. Every **block** can have a result, which means
that when the block's stack frame reaches the end of
its lifetime, the last item is popped. Then that frame is destroyed (i.e. the
mark is removed) and the popped item is pushed to the parent frame.
That's how functions work too, but they can have parameters and can
be executed at any time.

## Hello, world! Well, sort of.
WebAssembly Text files always start with the module definition and everything else is put between the `module` word and the last parenthesis. Let's see how we can write a simple "Hello, world!" program in WAT.
```wat
(module
	(func $hello_world (param $lhs i32) (param $rhs i32) (result i32)
		local.get $lhs
		local.get $rhs
		i32.add)
	(export "helloWorld" (func $hello_world)))
```
Okay, you might be saying "What the hell is this? I thought this is a 'Hello, world!' example!".
Well, the point is that WASM wasn't created to print strings and interact with
APIs, its purpose is to help JavaScript handle heavy computations by providing
an interface to fast, low level instructions.
### But what does the code do?
 - `i32` is one of the four primary types of WebAssembly, it's a 32-bit integer type.
 - `func` declares a function
 - `$hello_world` is a compile time name/label we give the function (we'll see more about that later)
 - `(param $lhs i32)` and `(param $rhs i32)` tell that this function accepts two parameters, the first one labeled $lhs for left-hand side (notice the `$`) with a type of `i32`,
 	the second one labeled $rhs for right-hand side with a type of `i32`.
 - `(result i32)` says that the function return type is an `i32`.
 - `local.get $lhs` pushes the value of the parameter labeled as $lhs into the stack.
 - `local.get $rhs` does the same as above, but instead it pushes the value of $rhs.
 - `i32.add` pops two values of type `i32` from the stack and pushes the result of their addition back into the stack.
 - `(export "helloWorld" (func $hello_world))` exports the function labeled as `$hello_world` to the host with the name "helloWorld".

### Where is the `return` statement?
WebAssembly has a `return` instruction, but it is only used when you need
to immediately return and stop executing the function any further. Otherwise,
there will be an implicit return at the end of the function that pops the last
value on the stack and returns it.

### What does a "label" actually mean?
All the function calls, parameter and local access are done by index. Labels
are just compile-time annotations to make code easier to read, write and
understand.

### Are `local.get` and `i32.add` the only instructions?
Of course not. The WebAssembly instruction set is *huge*. Just the MVP (minimal viable product)
has around 120 instructions. Most of them start with the type, which can only
be `i32`, `i64`, `f32`, `f64`, followed by a dot and the name of the
operation. There are also instructions for other purpouses, like control flow
(decision making, branching/jumping and looping).
A list of all the instructions and their explanation can be found [here](https://github.com/sunfishcode/wasm-reference-manual/blob/master/WebAssembly.md#instructions).

## Writing something useful
The example below shows how you can write a function that calculates the
distance between two points using Pythagorean theorem.
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
   If you don't understand the difference between signed and unsigned numbers, I suggest you read [this](https://dev.to/aidiri/signed-vs-unsigned-bit-integers-what-does-it-mean-and-what-s-the-difference-41a3)
 - `f64.sqrt` pops a number of type `f64` from the stack and pushes the
   number's square root.
 - `i32.mul` pops two numbers of type `i32` from the stack and pushes back the
   result of their multiplication.
   (we multiply a number with himself to get its power of 2 in the `$square` function).

## Globals
Globals are just "variables" that every function can access. 
They can be exported but can only be mutated if they are declared as mutable. 
The code below shows a way how they can be used:
```wat
(module
	(;
	  declare a mutable global with type `i32` and initial value 0
	;)
	(global $counter (mut i32) (i32.const 0))
	(export "counter" (global $counter)) ;; export it with the name "counter"
	(func $countUp (result i32)
		global.get $counter ;; push the value of $counter into the stack
		i32.const 1
		i32.add ;; increment the value by 1
		global.set $counter ;; assign $counter to the incremented value
		global.get $counter) ;; return the new incremented value
	(export "countUp" (func $countUp)))
```

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
The 3 first instructions are simple enough to be explained with comments, but
if you still don't understand what they do:
 - `local.get $0` and `local.get $1` push the values of the parameters labeled
   `$0` and `$1`
 - `i32.gt_s` pops two values of type `i32` from the stack and pushes back
   the result of their comparison (value of `param $0` is greater than the
   valule of `param $1`). `1` if true, `0` if false. It has the `_s` suffix
   because it compares the numbers as signed ones (i.e. they can be negative).

### To return `$0` or to return `$1`
When an `if` instruction occurs, the last item in the stack (the
condition) is popped. It must be an `i32`. If the condition is not `0`, the
instructions inside the `if..else/end` block are executed, otherwise the
`else..end` is executed. If there isn't an `else`, nothing happens.
You might notice that the if "statement" has a result. `if` is a **block**, 
which means it is able to return a result.

### Select
For simple decisions like picking a number, `if` might be a bit overkill.
There is also a `select` instruction which works like a ternary operator.
To use `select` instead of `if`, we would do it like this.
```wat
local.get $0
local.get $1 ;; push value of $0 and $1 into stack for selection
local.get $0
local.get $1 ;; push value of $0 and $1 into stack for comparing
i32.gt_s ;; doing the comparison between the two last `i32` items on the stack
select
```
The `select` instruction pops 3 values from the stack. It decides which of
the two first values to push back according to the third one (the condition). 
(if the condition is not `0`, push the first value, otherwise the second).

## Looping and branching
WebAssembly supports loops, but not the kind of loops you might be thinking
of. Take this code for example:
```wat
(module
	(func $fac (param $0 i32) (result i32) (local $acc i32)
		(local $num i32)

		local.get $0
		local.tee $acc
		local.set $num
		block $outer
			loop $loop
				local.get $num
				i32.const 1
				i32.lt_u ;; we check if $num is lower than 1
				br_if $outer
				local.get $acc
				local.get $num
				i32.const 1
				i32.sub
				local.tee $num
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

### Breaking down the code
Most of the things in this code have been explained. The only new things
here are `(local $acc i32)`, `block`, `loop`, `br_if $outer` and `$br $loop`.
 - `(local $acc i32)` is similar to what they call a local variable in higher level
   languages. It is accessed the same way as the parameters. The first
   local's index is 1 more than the last parameter's index.
 - `block $outer` does nothing special, it just encapsulates the code
   between it and the corresponding `end` instruction. It is used when you
   need to branch out at different levels.
 - `loop $loop` is a special kind of **block**, if you do a branch on a loop,
   you don't go at the `end`, you go to the beginning of the loop.
   (branches behave like `break` on a `block` and `if` and like `continue` on a `loop`)
 - `br $loop` is an unconditional branch. The label operand is a `loop`, so
   it will jump to the top where the `loop` instruction occurs. If you know
   C/C++ you know about jumping using `goto`, but by using `goto` you can jump
   anywhere, from anywhere in the code. WebAssembly is more restrictive in 
   the way that you can branch only outwards and by label / index. The innermost
   **block** has the smallest index (`0`) and the outermost has the largest.
   We do the branch so we can continue looping instead of dropping out.
 - `br_if $outer` is a conditional branch/jump instruction. It will pop the
   last item from the stack (the condition, has to be an `i32`) and if it is
   different from `0`, it will execute the branch, otherwise it won't.

## Linear memory
WebAssembly offers another way to store data other than the stack, the linear
memory. It can be seen as a resizable JavaScript `TypeArray`. Its main
purpose is to store complex and/or continous data. There are 14 load and
9 store instructions, and 2 other instructions for manipulating and getting
its size. With what we have learned until now, let's implement a function
that generates the fibonacci sequence.
```wat
(module
	(memory 1)
	(export "memory" (memory 0))
	(func $fib (param $length i32) (local $offset i32)
		i32.const 8
		local.set $offset ;; assign offset to 8 (see below)
		i32.const 0
		i32.const 1
		i32.store offset=4 ;; store 1 at offset 0 eith static offset 4
		block $block
			loop $loop
				local.get $offset
				i32.const 4
				i32.div ;; divide by 4(read below)
				local.get $length
				i32.gt_u ;; compare the requsted length
				br_if $block ;; break out if false (`9`)
				local.get $offset ;; get the offset for storing
				local.get $offset
				;; ---------
				i32.const 8
				i32.sub
				i32.load
				local.get $offset
				i32.const 4
				i32.sub
				i32.load
				i32.add ;; load the two previous numbers from memory and add them
				;; ---------
				i32.store ;; store the number at the current offset
				local.get $offset
				i32.const 4
				i32.add
				local.set $offset ;; increment the offset by 4
				br $loop
			end
		end)
	(export "fibonacci" (func $fib)))
```

### Explaining a few things
The code above uses almost all the knowledge that you have gained while
reading this article. A few things to note:
 - `store offset=4` stores an `i32`. It pops two items from the stack. The
   first one is the address/offset and the second is the value that will be
   stored. `offset=4` is a static offset, which means it will add that to the
   offset that it gets from the stack, without you needing to do
   `i32.const 4` and `i32.add` on the offset. In this example, `offset` was only 
   used for demonstrative purposes.
 - We use the offset instead of the index, because we don't have fancy things
   like arrays. Each `i32` takes 4 bytes and we have to increment the
   offset by `4` instead of `1`.
 - The offset variable points at the address in memory where the number that will
   result from the addition of the two previous numbers will be stored,
   thats why it is initially 8. Two `i32`s take 8 bytes in memory.
 - We divide the offset by 4 when comparing it, to have it like an "index".

## The end.
I hope that this article gave you a deeper understanding on how WebAssembly
works. A big thanks to [rom](https://github.com/romdotdog) who edited and
corrected my article a few times.
