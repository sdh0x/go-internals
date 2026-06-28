## Why using strconv instead of fmt for converting typical data types to string
alot of us use ```fmt.Sprint``` to convert basic data types to string, but it causes performance degradation especially if you are building a CLI utility

### fmt.Sprint Causes Heap Escapes
here is a code sample that checks if X is lower than 0 , if true it formates it as a complex number formula (x + yi) 
then x goes to math.Sqrt() function to square root

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	var x float64 = -5 
	var str string	
	if x < 0 {
		str = fmt.Sprint(math.Sqrt(-x)) + "i"
		fmt.Println(str)
	} else {
		fmt.Println(math.Sqrt(x))
	}
}
```
we need then to convert the **float64** to **string** to show case why you should use **strconv*** instead of **fmt**, we used ```fmt.Sprint```, used for converting basic data types to string, this is why we called fmt twice, one for converting and one for printing the output

* If we compile with the flag '-m' for the golang compiler which stands for escape analysis to instruct the compiler to print out its optimization decisions, what escaped to the heap and what stayed in the stack frame of the function

we can see that the variable str escaped to the heap besides that all in stack frame of the main function 

```bash
[lost main]$ go build -gcflags='-m' main.go
# command-line-arguments
./main.go:12:29: inlining call to math.Sqrt
./main.go:13:14: inlining call to fmt.Println
./main.go:15:24: inlining call to math.Sqrt
./main.go:15:14: inlining call to fmt.Println
./main.go:12:35: fmt.Sprint(... argument...) + "i" escapes to heap
./main.go:12:19: ... argument does not escape
./main.go:12:29: ~r0 escapes to heap
./main.go:13:14: ... argument does not escape
./main.go:13:15: str escapes to heap
./main.go:15:14: ... argument does not escape
./main.go:15:24: ~r0 escapes to heap
```

str did escape to the heap, we dont need that, heap causes overload.

### Using strconv

strconv is the best alternative solution with a built in function for printing text to the terminal ``` println(str) ``` 

```go
package main

import (
	"math"
	"strconv"
)

func main() {
	var x float64 = -5
	var str string
	if x < 0 {
		str = strconv.FormatFloat(math.Sqrt(-x), 'f', -1, 64) + "i" // 'f' stands for no exponent, take a look at strconv doc in pkg.go.dev/strconv
		println(str)
	} else {
		println(strconv.FormatFloat(math.Sqrt(x), 'f', -1, 64))
	}

}
```

if we compile with the same flag for showing the compiler optimizing decisions 

```bash
[lost main]$ go build -gcflags='-m' main.go
# command-line-arguments
./main.go:12:38: inlining call to math.Sqrt
./main.go:12:28: inlining call to strconv.FormatFloat
./main.go:15:20: inlining call to math.Sqrt
./main.go:12:28: inlining call to strconv.FormatFloat
./main.go:12:57: ~r0 + "i" does not escape
./main.go:12:28: string(strconv.genericFtoa(make([]byte, 0, max(strconv.prec + 4, 24)), strconv.f, strconv.fmt, strconv.prec, strconv.bitSize)) does not escape
./main.go:12:28: make([]byte, 0, max(strconv.prec + 4, 24)) does not escape
```
as you can see, there is no "Escaped to the heap" there is only does not escape

## why is that ? 
fmt.Sprint takes an empty interface{} which has two-pointer structure (eface): one pointer to the dynamic type information, and one pointer to the actual data, when you pass a concrete type (float64) into fmt.Sprint go wraps it into this interface structure.

Devirtualization: This is an optimization method where the compiler attempts to look through the interface, determine the concrete type then call the concrete method directly.

Failure to Devirtualize: Because fmt passes your variable down a deeply nested chain of internal functions that rely on reflection, the escape analysis algorithm completely loses visibility; ( Escape analyses is an optimization method where the compiler chooses what stays in the stack and what escapes to the heap ) it cant trace the pointers lifecycle across these abstract interface boundaries

As Safety Net the compiler cannot prove that a variable will not outlive the stack frame of the function so it cannot safely leave it on the stack, as a strict memory safety-net decision by the compiler, the variable escapes to the heap.
