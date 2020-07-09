# TECHNIQUE 10 Using goroutine closures
Any function can be executed as a goroutine. And because Go allows you to declare
functions inline, you can share variables by declaring one function inside another and
closing over the variables you want to share.
## PROBLEM
You want to use a one-shot function in a way that doesn’t block the calling function,
and you’d like to make sure that it runs. This use case frequently arises when you want
to, say, read a file in the background, send messages to a remote log server, or save a
current state without pausing the program.
## SOLUTION
Use a closure function and give the scheduler opportunity to run.
## DISCUSSION
In Go, functions are first-class. They can be created inline, passed into other functions,
and assigned as values to a variable. You can even declare an anonymous function and
call it as a goroutine, all in a compact syntax, as the following listing shows.
```go
package main
import (
    "fmt"
    "runtime"
)
func main() {
    fmt.Println("Outside a goroutine.")
    go func() {
        fmt.Println("Inside a goroutine")
    }()
    fmt.Println("Outside again.")
    runtime.Gosched()
}
```
This listing shows how to create the function inline and immediately call it as a gorou-
tine. But if you execute this program, you may be surprised at the output, which may
change from run to run. It’s not uncommon to see this:
```sh
$ go run ./simple.go
Outside a goroutine.
Outside again.
Inside a goroutine
```
Goroutines run concurrently, but not necessarily in parallel. When you schedule a
goroutine to run by calling go func , you’re asking the Go runtime to execute that
function for you as soon as it can. But that’s likely not immediately. In fact, if your Go
program can use only one processor, you can almost be sure that it won’t run immedi-
ately. Instead, the scheduler will continue executing the outer function until a circum-
stance arises that causes it to switch to another task. This leads us to another facet of
this example.
You may have noticed the last line of the function, runtime.Gosched() . This is a
way to indicate to the Go runtime that you’re at a point where you could pause and
yield to the scheduler. If the scheduler has other tasks queued up (other goroutines),
it may then run one or more of them before coming back to this function.
If you were to omit this line and rerun the example, your output would likely look
like this:
```sh
$ go run ./simple.go
Outside a goroutine.
Outside again.
```
The goroutine never executes. Why? The main function returns (terminating the pro-
gram) before the scheduler has a chance to run the goroutine. When you run runtime
.Gosched , though, you give the runtime an opportunity to execute other goroutines
before it exits.
There are other ways of yielding to the scheduler; perhaps the most common is to
call time.Sleep . But none gives you the explicit ability to tell the scheduler what to do
when you yield. At best, you can indicate to the scheduler only that the present gorou-
tine is at a point where it can or should pause. Most of the time, the outcome of yield-
ing to the scheduler is predictable. But keep in mind that other goroutines may also
hit points at which they pause, and in such cases, the scheduler may again continue
running your function.
For example, if you execute a goroutine that runs a database query, running
runtime.Gosched may not be enough to ensure that the other goroutine has com-
pleted its query. It may end up paused, waiting for the database, in which case the
scheduler may continue running your function. Thus, although calling the Go sched-
uler may guarantee that the scheduler has a chance to check for other goroutines,
you shouldn’t rely on it as a tool for ensuring that other goroutines have a chance to
complete.
There’s a better way of doing that. The solution to this complex situation is shown
next.