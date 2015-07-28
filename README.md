Benchmarks.jl
=============

A package to make Julia benchmarking both easy and rigorous. See the design
section for details about what we mean by rigor.

# Basic Usage Example

For trivial benchmarking, you can use the `@benchmark` macro, which takes
in a simple Julia expression and returns the results of benchmarking that
expression:

```jl
using Benchmarks

@benchmark sin(2.0)

@benchmark sin(sin(2.0))

@benchmark exp(sin(2.0))

@benchmark svd(rand(10, 10))
```

# Advanced Usage Example

For more advanced usage (and for all automated usage), you need to use a
lower-level interface than the `@benchmark` macro. Here we give a minimal
example of that lower-level interface:

```jl
import Benchmarks

Benchmarks.@benchmarkable(
    sin_benchmark!,
    nothing,
    sin(2.0),
    nothing
)

r = Benchmarks.execute(sin_benchmark!)
```

In the design section, we'll describe the way that the lower-level interface
works.

# The Design of the Benchmarks Package

Benchmarking is hard. To do it well, you have to care about the details of how
code is executed and the statistical challenges that make it hard to generalize
correctly from benchmark data. To explain the rationale for design of the
package, we discuss the problems that the package is designed to solve.

## Measurement Error: Benchmarks that Measure the Wrong Thing

The first problem that the Benchmarks package tries to solve is the problem of
measurement error: benchmarking any expression that can be evaluated faster
than the system clock's resolution can track is vulnerable to measuring the
system clock's performance rather than the expression's performance. Naive
estimates, like those generated by the `@time` macro are often totally
inaccurate. To convince yourself, consider the following timings:

```j
@time sin(2.0)

@time sin(sin(2.0))

@time sin(sin(2.0))
```

On my system, the results of this code often suggest that `sin(sin(2.0))` can
be evaluated faster than `sin(2.0)`. That's obviously absurd -- and the timings
returned by `@time sin(2.0)` are almost certainly totally inaccurate.

The reason for that inaccuracy is that `sin(2.0)` can be evaluated much faster
than the system clock's finest resolution. As such, you're almost exclusively
measuring variability in the system clock when you evaluate `@time sin(2.0)`.

To deal with this, the Benchmarks package exploits a simple linear
relationship: evaluating an expression N times in a row should take
approximately N times as long as evaluating the same expression exactly 1
time. This linear relationship holds almost perfectly as N grows. Thus, we
solve the problem of measurement error by measuring the time it takes to
evaluate `sin(2.0)` a very large number of times. Then we apply linear
regression to estimate the time it takes to evaluate `sin(2.0)` exactly once.

# Accounting for Variability

When you repeatedly evaluate the same expression, you find that the timings
you measure are not all the same. Instead, there is often substantial
variability across measurements.

Traditional statistical methods are often employed to resolve this problem. The
Benchmarks package tries to estimate the average time that it would take to
evaluate an expression. Because the average is estimated from a small sample
of measurements, we have to acknowledge that our estimate is uncertain. We
do this by reporting 95% confidence intervals (hereafter called CI's) for
the average time per evaluation.

# Constrained Budgets

Getting the best estimate of the average time requires gathering as many
samples as possible. Most people want their benchmarks to run in a finite
amount of time, so we need to impose some budget. The Benchmarks package
exposes both a sample budget and a time budget. For very long computations,
the time budget prevents the benchmarking process from taking more than 10s.
This can be configured.

For mid-range computations, we also allow users to insist on acquiring at most
100 samples. This can be effective if the time budget seems excessive for
the purpose of estimating CI's accurately.

For very fast computations that require OLS modeling, we ignore the samples
budget, although we respect the user's specificed time budget. This is because
users cannot reasonably expect to know how many samples they need to gather
to estimate the average time accurately.

# Dependence on External Resources

Sometimes we want to benchmark functions that depend upon external resources.
For example, we want to time how long it takes us to parse a file, which we
must first download.

To handle these scenarios, the `@benchmarkable` macro takes in three
expressions:

* `setup`: This expression will be evaluated once before the benchmark starts
      executing.
* `core`: This expression is the core expression we want to benchmark.
* `teardown`: This expression will be evaluated once after the core expression
      stops executing.