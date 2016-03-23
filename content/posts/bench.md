+++
title = "Go Benchmarks"
description = "How to write benchmarks and analyze their result"
date = "2015-09-24"
categories = ["go"]
+++

## Benchmarks

Benchmarks are tests for performance. It's pretty useful to have them in
project and compare results from commit to commit. Go has very good tooling for
writing and executing benchmarks. In this article I'll show how to use package
`testing` for writing benchmarks.

## How to write benchmark

It's pretty easy in Go. Here is a simple benchmark:

{{< highlight go >}}
func BenchmarkSample(b *testing.B) {
    for i := 0; i < b.N; i++ {
        if x := fmt.Sprintf("%d", 42); x != "42" {
            b.Fatalf("Unexpected string: %s", x)
        }
    }
}
{{< /highlight >}}

Save this code to `bench_test.go` and run `go test -bench=. bench_test.go`.
You'll see something like this:

```
testing: warning: no tests to run
PASS
BenchmarkSample 10000000               206 ns/op
ok      command-line-arguments  2.274s

```

We see here that one iteration takes 206 nanoseconds. That was easy, indeed.
There are couple of things more about benchmarks in Go, though.

## What you can benchmark?

By default `go test -bench=.` tests only speed of your code, however you can
add flag `-benchmem`, which will also test a memory consumption and an
allocations count. It'll look like:

```
PASS
BenchmarkSample 10000000               208 ns/op              32 B/op          2 allocs/op

```

Here we have bytes per operation and allocations per operation. Pretty useful
information as for me. You can also enable those reports per-benchmark with
`b.ReportAllocs()` method.
But that's not all, you can also specify a throughput of one operation with 
`b.SetBytes(n int64)` method. For example:

{{< highlight go >}}
func BenchmarkSample(b *testing.B) {
    b.SetBytes(2)
    for i := 0; i < b.N; i++ {
        if x := fmt.Sprintf("%d", 42); x != "42" {
            b.Fatalf("Unexpected string: %s", x)
        }
    }
}
{{< /highlight >}}

Now output will be:

```
testing: warning: no tests to run
PASS
BenchmarkSample  5000000               324 ns/op           6.17 MB/s          32 B/op          2 allocs/op
ok      command-line-arguments  1.999s
```

You can see now throughput column, which is `6.17 MB/s` in my case.

## Benchmark setup

What if you need to prepare your operation for an each iteration? You definitely
don't want to include time of setup in a benchmark result.
I wrote very simple `Set` datastructure for benchmarking:

{{< highlight go >}}
type Set struct {
    set map[interface{}]struct{}
    mu  sync.Mutex
}

func (s *Set) Add(x interface{}) {
    s.mu.Lock()
    s.set[x] = struct{}{}
    s.mu.Unlock()
}

func (s *Set) Delete(x interface{}) {
    s.mu.Lock()
    delete(s.set, x)
    s.mu.Unlock()
}
{{< /highlight >}}
and benchmark for its `Delete` method:
{{< highlight go >}}
func BenchmarkSetDelete(b *testing.B) {
    var testSet []string
    for i := 0; i < 1024; i++ {
        testSet = append(testSet, strconv.Itoa(i))
    }
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        set := Set{set: make(map[interface{}]struct{})}
        for _, elem := range testSet {
            set.Add(elem)
        }
        for _, elem := range testSet {
            set.Delete(elem)
        }
    }
}
{{< /highlight >}}

Here we have couple of problems:

* time and allocs of `testSet` creation included in first iteration (which isn't 
  big problem here, because there will be a lot of iterations).
* time and allocs of `Add` to set included in each iteration

For such cases we have `b.ResetTimer()`, `b.StopTimer()` and `b.StartTimer()`.
Here those methods used in same benchmark:

{{< highlight go >}}
func BenchmarkSetDelete(b *testing.B) {
    var testSet []string
    for i := 0; i < 1024; i++ {
        testSet = append(testSet, strconv.Itoa(i))
    }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        set := Set{set: make(map[interface{}]struct{})}
        for _, elem := range testSet {
            set.Add(elem)
        }
        b.StartTimer()
        for _, elem := range testSet {
            set.Delete(elem)
        }
    }
}
{{< /highlight >}}

Now those initializations won't be counted in benchmark results and we'll see
only results of `Delete` calls.

## Benchmarks comparison

Of course there is nothing to do with benchmark if you can't compare them on
different code.

Here is an example code of marshaling struct to json and benchhmark for it:

{{< highlight go >}}
type testStruct struct {
    X int
    Y string
}

func (t *testStruct) ToJSON() ([]byte, error) {
    return json.Marshal(t)
}

func BenchmarkToJSON(b *testing.B) {
    tmp := &testStruct{X: 1, Y: "string"}
    js, err := tmp.ToJSON()
    if err != nil {
        b.Fatal(err)
    }
    b.SetBytes(int64(len(js)))
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        if _, err := tmp.ToJSON(); err != nil {
            b.Fatal(err)
        }
    }
}
{{< /highlight >}}

It's commited in `git` already, now I want to try cool trick and measure its
performance. I slightly modify `ToJSON` method:

{{< highlight go >}}
func (t *testStruct) ToJSON() ([]byte, error) {
    return []byte(`{"X": ` + strconv.Itoa(t.X) + `, "Y": "` + t.Y + `"}`), nil
}
{{< /highlight >}}

Now it's time to run our bechmarks, let's save their results in files this time:

```
go test -bench=. -benchmem bench_test.go > new.txt
git stash
go test -bench=. -benchmem bench_test.go > old.txt
```

Now we can compare those results with
[benchcmp](https://godoc.org/golang.org/x/tools/cmd/benchcmp) utility. You can
install it with `go get golang.org/x/tools/cmd/benchcmp`. Here is result
of comparison:

```
# benchcmp old.txt new.txt
benchmark           old ns/op     new ns/op     delta
BenchmarkToJSON     1579          495           -68.65%

benchmark           old MB/s     new MB/s     speedup
BenchmarkToJSON     12.66        46.41        3.67x

benchmark           old allocs     new allocs     delta
BenchmarkToJSON     2              2              +0.00%

benchmark           old bytes     new bytes     delta
BenchmarkToJSON     184           48            -73.91%
```

It's very good to see such tables, they also can add weight to your opensource
contributions.

## Writing profiles

Also you can write cpu and memory profiles from benchmarks:

```
go test -bench=. -benchmem -cpuprofile=cpu.out -memprofile=mem.out bench_test.go
```

You can read how to analyze profiles in awesome blog post on blog.golang.org
[here](http://blog.golang.org/profiling-go-programs).

## Conclusion

Benchmarks is awesome instrument for programmer. And in Go you to writing and
analyzing becnhmarks is extremely easy. New benchmarks allows you to find
performance bottlenecks, weird code (efficient code is often simpler and more
readable) or usage of wrong instruments. Old benchmarks allow you to be more
confident in your changes and could be another +1 in review process. So,
writing writing benchmarks has enormous benefits for programmer and code and
I encourage you to write more. It's fun!
