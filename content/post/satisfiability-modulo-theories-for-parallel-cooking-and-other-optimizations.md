+++
Categories = ["Constraint Programming"]
Description = "Satisfiability Modulo Theories For Parallel Cooking And Other Optimizations"
Tags = ["SMT", "SAT", "Z3", "nuZ"]
date = "2015-05-08T20:34:36+02:00"
title = "Satisfiability Modulo Theories For Parallel Cooking And Other Optimizations"

+++

Using an optimizing constraint solver to answer fundamental questions, such as "How to optimize parallel Pizza cooking", and "How to optimize layouting in CSS".
In this article I will not only give answers to those questions but you will also get an introduction into Satisfiability (SAT) and Satisfiability Modulo Theories (SMT) solvers -- Z3 in particular.


## Motivation

Another week, another newly open sourced project to play with!
This time it is the [Z3 Theorem Prover](https://github.com/Z3Prover/z3) that I want to have fun with.

Let's get you started with the basics required for understanding the examples (read: you can skip till the "Parallel Cooking" section if you want code now).


## Satisfiability

Suppose we have a formula consisting of the *literals* A, B, C and a conjunction of two *clauses*, i.e. a formula in Conjunctive Normal Form (CNF):

```python
(A or C) and (not B)
```

Is there an assignment for the literals A, B and C to make the formula true?
Sure, I can come up with an assignment:

```python
{A: false, B: false, C: true}
```

Can you think of another one?

In general, this is the problem of satisfiability (SAT) which is NP-complete (read: hard).
Fortunately there are SAT-solvers that help us determine if such an assignment exists.
Most if not all SAT-solvers work with the DIMACS format that standardizes how to encode the formula.
The syntax is as follows:

```
c Comment
p cnf numberOfLiterals numberOfClauses
...
```

What follows after the p-line is a way to encode your formula:
literals are numbered starting from one.
A negative number means negation.
The zero at the end of the clause is a line terminator.

Our formula from above has three *literals* (A, B, C) and two *clauses* ((A or C), (not B)), the first clause consists of A (1), C (3), the second clause consists of not B (-2):

```
c this is a dimacs file
p cnf 3 2
1 3 0
-2 0
```

Let's feed this into Z3 and see what it comes up with:

```
sat
1 -2 -3
```

Which translates to:

```python
{A: true, B: false, C: false}
```

Here is an assignment for you: encode this into the DIMACS format and check for satisfiability:

```python
(A or B) and ((not B) or C or (not D)) and (D or (not E))
```

Let's go on with building abstraction on top of SAT solvers, because this can't be it!


## Satisfiability Modulo Theories

Now that we can encode problems for SAT solvers you may think:
"I can encode integers as boolean literals (read: in binary), much as my computer does it" and this is exactly what Satisfiability Modulo Theories (SMT) is all about:
an abstraction on top of SAT, that "knows" about so called theories.
Here are some theories:

* Theory of Integers: mathematical integers, not machine types
* Theory of Reals: mathematical reals, not machine type
* Theory of Bitvectors: vector of bits, for signed and unsigned two's-complement calculations

In addition, SMT also "knows" about their rules, e.g. how to add two 32 bit unsigned integers represented as so called Bitvectors.
This takes away the burden to manually implement unsigned addition for 32 boolean literals representing an integer in SAT.

As with the DIMACS format for SAT, there is a standardized format for SMT, the SMT-LIB 2.0 format.
It may look rather Lisp'ish -- but don't despair, it's still easy to understand (if not, learn yourself some Clojure for great good -- or use Z3's language bindings, e.g. the Python one).

Let's see if we can find an assignment for two integers, satisfying the constraints that x is ten and y should be greater than zero and less then ten:

```clojure
(set-option :produce-models true)
(set-logic QF_LIA)

(declare-fun x () Int)
(declare-fun y () Int)

(assert (= x 10))
(assert (< 0 y 10))

(check-sat)

(get-value (x))
(get-value (y))

(exit)
```

Z3 comes up with the following:

```
sat
((x 10))
((y 1))
```

Cool!
But wait, this only gives us a single solution or simply tells us "unsat".
What if we want an "optimal" solution (for our understanding of optimal) under certain constraints?


## An Optimizing SMT Solver

Z3's opt branch implements an optimizing SMT solver, called nuZ.
You can learn more about it in these two papers:

* [nuZ - An Optimizing SMT Solver](http://research.microsoft.com/en-US/people/nbjorner/nuz.pdf)
* [nuZ - Maximal Satisfaction with Z3](http://research.microsoft.com/en-US/people/nbjorner/scss2014.pdf)

Let's start with an example:
we want to maximize the sum of two integers under the constraints that the integer's values should be between zero and ten:

```clojure
(declare-const x Int)
(declare-const y Int)

(assert (<= 0 x 10))
(assert (<= 0 y 10))

(maximize (+ x y))

(check-sat)
(get-model)
```

Feeding this to Z3 gives us an optimal solution, maximizing the sum and also producing a model, showing x and y:

```clojure
(+ x y) |-> 20
sat
(model
  (define-fun y() Int 10)
  (define-fun x() Int 10))
```

Great! This is good enough for now, let's optimize something!


## Parallel Cooking

Say you want to cook Pizza. With multiple cooks. In parallel.
Can we come up with a schedule that minimizes the overall time it takes, while repecting constraints such as "We have to slice onions before putting the Pizza into the oven"?

Instead of working only with integers let's create an interval type that abstracts, well, an interval:

```clojure
(declare-datatypes (BeginT EndT) ((IntervalT (makeInterval (begin BeginT) (end EndT)))))
(define-sort Interval () (IntervalT Int Int))
```

This is rather ugly and I have to confess, I had to look up the exact syntax.
But anyway, now we have an interval type (what is called Sort by the SMT people) with constructor *makeInterval*, and  *begin* and *end* accessors.

Let's tell Z3 more about intervals.
Intervals expand into the future, and have a duration.

```clojure
(define-fun intervalExpandsIntoFuture? ((interval Interval)) (Bool)
  (< (begin interval) (end interval)))

(define-fun intervalDuration ((interval Interval)) (Int)
  (- (end interval) (begin interval)))
```

If an interval requires a sescond interval to be over, this can also be expressed.
The same goes for the requirement of an interval being during a second interval:

```clojure
(define-fun intervalRequiresIntervalOver ((lhs Interval) (rhs Interval)) (Bool)
  (> (begin lhs) (end rhs)))

(define-fun intervalDuringInterval? ((lhs Interval) (rhs Interval)) (Bool)
  (and
    (> (begin lhs) (begin rhs))
    (< (end lhs) (end rhs))))
```

Let's create an overall schedule its duration we want to minimize in the end.
And also create intervals for tasks such as making the dough, slicing onions, slicing tomatos, and so on:

```clojure
(declare-const schedule Interval)

(declare-const makeDough Interval)
(declare-const sliceOnions Interval)
(declare-const sliceTomatos Interval)
(declare-const bakeInOven Interval)
(declare-const servePizza Interval)
(declare-const pourWine Interval)
```

Now for the constraints, this is where we tell Z3 what properties we require in a solution.
Here we tell Z3 that our schedule starts at time zero, intervals do not go back in time, our tasks should fall into the overall schedule, estimated durations for the tasks and finally the task's dependencies;

```clojure
(assert (and
          (= 0 (begin schedule))
          (intervalExpandsIntoFuture? schedule)
 
          (intervalExpandsIntoFuture? makeDough)
          (intervalExpandsIntoFuture? sliceOnions)
          (intervalExpandsIntoFuture? sliceTomatos)
          (intervalExpandsIntoFuture? bakeInOven)
          (intervalExpandsIntoFuture? servePizza)
          (intervalExpandsIntoFuture? pourWine)
 
          (intervalDuringInterval? makeDough schedule)
          (intervalDuringInterval? sliceOnions schedule)
          (intervalDuringInterval? sliceTomatos schedule)
          (intervalDuringInterval? bakeInOven schedule)
          (intervalDuringInterval? servePizza schedule)
          (intervalDuringInterval? pourWine schedule)
 
          (< 3 (intervalDuration makeDough))
          (< 2 (intervalDuration sliceOnions))
          (< 2 (intervalDuration sliceTomatos))
          (= 35 (intervalDuration bakeInOven))
          (< 2 (intervalDuration servePizza))
          (< 2 (intervalDuration pourWine))
 
          (intervalRequiresIntervalOver bakeInOven makeDough)
          (intervalRequiresIntervalOver bakeInOven sliceOnions)
          (intervalRequiresIntervalOver bakeInOven sliceTomatos)
 
          (intervalRequiresIntervalOver servePizza bakeInOven))
 
(minimize (intervalDuration schedule))
```

Boom!

```clojure
(- (end schedule) (begin schedule)) |-> 46
sat
(model
  (define-fun sliceTomatos () (IntervalT Int Int) (makeInterval 1 4))
  (define-fun pourWine () (IntervalT Int Int) (makeInterval 36 39))
  (define-fun schedule () (IntervalT Int Int) (makeInterval 0 46))
  (define-fun servePizza () (IntervalT Int Int) (makeInterval 42 45))
  (define-fun sliceOnions () (IntervalT Int Int) (makeInterval 1 4))
  (define-fun makeDough () (IntervalT Int Int) (makeInterval 1 5))
  (define-fun bakeInOven () (IntervalT Int Int) (makeInterval 6 41)))
```

Z3 wants us to make the dough in parallel with slicing tomatos and onions.
We then have to bake the Pizza, and before it is done pour the wine; a few minutes later we serve the Pizza.
Enjoy!


Note: although this is an optimal schedule we certainly want to improve the constraints , e.g. you want to specify the number of cooks -- but 70 lines of code is a good point to stop with an example.



## Further Remarks

Pandoc-generated notes are available [here](https://gist.github.com/daniel-j-h/09c772b706db7a7240a4).
The parallel cooking code can be found [here](https://gist.github.com/daniel-j-h/09c772b706db7a7240a4#file-9-parallelize-clj).

If you are still reading this, [here](https://gist.github.com/daniel-j-h/09c772b706db7a7240a4#file-8-layouting-clj) is the promised layouting example including CSS from a solution, which is a bit more elaborate.

See the official [nuZ tutorial](http://rise4fun.com/Z3Opt/tutorial/) for more details.

Note: beware issues in Z3/nuZ, [I reported one](https://github.com/Z3Prover/z3/issues/52) during working on the parallel cooking example.


## Summary

In preparation for optimizing parallel Pizza cooking you hopefully learnt something about Satisfiability (SAT), Satisfiability Modulo Theories (SMT) and its applications.
We built an interval abstraction and specified constraints about interval durations and dependencies for cooking tasks.
Using Z3's opt branch (nuZ) we then minimized the overall time it takes to cook and serve the Pizza.


## Comment

Join the discussion over at [HackerNews]() or Reddit's [/r/programming]().
