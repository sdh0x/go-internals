## Why using strconv instead of fmt for converting typical data types to string
alot of us use ```fmt.Sprint``` to convert basic data types to string, but it causes performance degradation

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
	var x float64 = 5 
	var str string	
	if x < 0 {
		str = fmt.Sprint(math.Sqrt(-x)) + "i"
		fmt.Println(str)
	} else {
		fmt.Println(math.Sqrt(x))
	}
}
```
we need then to convert the **float64** to **string** to show case why you should use **strconv*** instead of **fmt**, we used ```fmt.Sprint```, it wont print anything to the terminal, its used for converting basic data types to string, this is why we called fmt twice, one for converting and one for printing the output

* but here is the catch, if we compile with the flag '-m' for the golang compiler which stands for escape analyses to instruct the compiler to print out its optimization desicions, what escaped to the heap and which stayed in the stack frame of the function

we can see that the variable str escaped to the heap besides that all in stack frame of the main function 

```bash
[linux main]$ go build -gcflags='-m' main.go
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

str did escape to the heap, we dont need that, heap causes overload especially if your app is cli based utility
strconv is the best alternative solution to this, take look at that code example > note : we used built it function for printing to the terminal ``` println(str) ``` 

```go
package main

import (
	"math"
	"strconv"
)

func main() {
	var x float64 = 5
	var str string
	if x < 0 {
		str = strconv.FormatFloat(math.Sqrt(-x), 'f', -1, 64) + "i"
		println(str)
	} else {
		println(math.Sqrt(x))
	}

}
```

if we compile with the same flag for showing the compiler optimizing decisions 
```bash
[linux main]$ go build -gcflags='-m' main.go
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

why is that ? 
its because fmt function when handling your input "variable we used" it is so nested into other multiple functions that the compiler cant trace end to end if that will be recalled "the variable" , and this is called deep call chain method  **Method devirualization** and because it evaluates this through an  **interface{}** , the compiler encounters a limitation called **failure to devirtualize** , so to not crach the application & computer , the compiler makes the var escapes to the heap as a safety net.
