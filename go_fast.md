# go fast

How to write fast code on golang.

## Test

Before you start your code better to be tested.
It prevents you from optimizing something working not as it must do.

It's also a good practice to have a good coverage first. It'll help you not to break your code while optimizing.

```bash
go test

cover # opens cover profile
```
```bash
# .bashrc
cover() {
	f=/tmp/cover.out
	rm -f $f
	go test "$@" -coverprofile $f
	[ $(cat /tmp/cover.out | wc -l) -gt 1 ] && go tool cover -html $f
}
```

## Measure

Do not spend time on premature optimizations, measure first.

First thing to do is

```bash
go test # check if everything works fine
go test -c -o run.test # save executable for better code annotation (used by pprof).
./run.test - -test.bench <one_func_in_a_time_here> -test.benchmem -memprofile run.mprof

# what you'll see
BenchmarkTlogTracesConsoleDetailed-8      	  429392	      2729 ns/op	      24 B/op	       2 allocs/op
```

## Allocs

0 is the maximum allowed number of allocs for a hotpath.

Check where you do yours.
```bash
# start from -alloc_objects and remove some allocs totally
go tool pprof -alloc_objects run.mprof # executable is found automatically
# ...
(pprof) top
# ...
(pprof) list <top1_function_name>
# ...
(pprof) weblist <your_package_name>
```

inspect -> fix -> repeat

When you can't avoid any more allocs go to the next step. Try to minimize objects you allocate.
```bash
go tool pprof -alloc_space run.mprof
(pprof) top
(pprof) list <top1_function_name>
(pprof) weblist <your_package_name>
```

### Patterns to avoid allocs

* Reuse buffers
* Avoid pointers exept:
    - There are o(1) of these objects in program
    - Object state changes during operation by design
    - Object is hondreds of bytes

Even if you work with only one goroutine all data is treated locally
in some situations go move data to heap just in case.
* sending and receiving data to and fron channel
* callcack functions with closure

golangci-lint maligned linter detects structures you may optimize without any changes in application logic.

There is way to inspect which variables (and possibly why) escape to heap and so produce allocs.
```bash
go build -gcflags '-m -m' # and more -m
```

## CPU

### Heap and Stack

Even if you get rid of all your allocs treading memory right still can give you some nanoseconds.
Strive to keep your data small and grouped by usage. Keywords here is data locality and CPU cache.
Usual CPU first level cache is 64 bytes.

Shorter the code -> fewer instructions needed to store it -> fewer memory reads ->
better cache utilization -> faster program works.

Cache optimization may result in x2 gain in speed.

### `b[i] = v; i++; copy(b[i:], str) vs b = append(b, v)`

I have been thinking tnat first is faster: there is no function call, just two simple operations.
Practice clarified that it's vice versa: each indexing or slicing involves boundary checks.
First approach is also usually longer and harder to read and undersnand.

### Inlining

Function with no calls to another functions and not longer than 80 instructions is inlined.
It saves instructions for `call`, helps to keep CPU cache in place.
Function that is just a call of another function could be inlined either.

### defer mu.Unlock()

```go
mu.Lock()
defer mu.Unlock()

vs

defer mu.Unlock()
mu.Lock()
```
`defer` costs 12 instructions.
In the first case they are executed under lock in the second - not. It's a little but it could be noticeable; up to 2-3%.

## Dirty hacks

### `//go:linkname`
Suppose you use 3rd party or stdlib function that makes allocs.
You open code and see that it could be avoided.
Functions is too big and/or relies on private package members or this function is private itself.
Another case is when you need only part of the work this functions does.
You may copy some small part of code to your library and change it to meet your needs.
Now you need to resolve calls to foreign private functions.
Solution is `//go:linkname`

Examples
* Stdlib: https://github.com/golang/go/blob/release-branch.go1.13/src/time/time.go#L1081
* tlog: https://github.com/nikandfor/tlog/blob/v0.4.0/unsafe.go#L65

### `//go:noescape`

Sometimes previous hack needs a little help.
To prevent arguments of functions linked by `//go:linkname` to escape to heap
(go can't analyze it without function body. And it does it before linking.) you can mark this function as
`//go:noescape`

Example: https://github.com/nikandfor/tlog/blob/v0.4.0/fmt.go#L74

### `noesacape`

I can't come up with general and simple example when you need this.
But if you need you'll know.

```go
// go/src/runtime/stubs.go

// noescape hides a pointer from escape analysis.  noescape is
// the identity function but escape analysis doesn't think the
// output depends on the input.  noescape is inlined and currently
// compiles down to zero instructions.
// USE CAREFULLY!
//go:nosplit
func noescape(p unsafe.Pointer) unsafe.Pointer {
    x := uintptr(p)
    return unsafe.Pointer(x ^ 0)
}
```

Example in tlog.
Without this hack p was escaping to heap and I didn't realized the reason.
https://github.com/nikandfor/tlog/blob/v0.4.0/fmt.go#L86
