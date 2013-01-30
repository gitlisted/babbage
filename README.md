babbage
=======

A library to create computation engines.

## Usage

```clojure
[babbage "0.1.0"]

;; In your ns statement:

(ns my.ns 
    (:use babbage.core))
```

## Introduction

The basic interface is provided by the functions `stats`, `sets`, and
`calculate`. 

`stats` is used to [declare the measures](#multiple-stats-) to calculate.

`sets` is used to [declare the subsets](#multiple-subsets-) of the input over which the
measures should be calculated.

`calculate` is used to actually make the thing go: it takes an
optional set specification as returned by `sets`, a measure
specification as returned by `stats` (or a map whose values are such
specifications), and a seq of inputs to calculate measures over.

```clojure
;; Calculate the sum of a seq of elements.
user> (require '[babbage.provided.core :as b])
user> (calculate
        (stats identity b/sum) ;; A stat that's the sum of the elements.
        [1 2 3 4])
{:all {:sum 10}} ;; :all refers to the result computed over all elements, not a subset
```

Also provided is a mechanism to perform [efficient computation over directed graphs](#efficient-computation-of-inputs-).

## Multiple stats

The `stats` function takes an *extraction function* as its first
argument, and an arbitrary number of *measure functions* as its
remaining arguments.

In the simplest possible case, as we saw above, the extraction function is identity:

```clojure
;; Calculate the sum of a seq of elements.
user> (calculate (stats identity b/sum) [1 2 3 4])
{:all {:sum 10}} ;; :all refers to the result computed over all elements, not a subset
```

Frequently we will want to both name the result, and perform some kind
of operation on the elements of the input:

```clojure
user> (calculate 
        {:the-result (stats :x b/sum)} ;; Compute the sum over :x's. Call it ":the-result".
        [{:x 1} {:x 2}])
{:all {:the-result {:sum 3}}} 
```

Multiple measures can be computed in one pass:

```clojure
user> (->> [{:x 1} {:x 2}] 
     (calculate {:the-result (stats :x b/sum b/mean)})) ;; <-- Here we've added the mean as well.
{:all {:the-result {:mean 1.5, :count 2, :sum 3}}} ;; The 'count' measure is required by 'mean', 
                                                   ;; so it's automatically computed.
```

And we can compute multiple measures over multiple fields in one pass:

```clojure
;; Compute multiple measures over multiple fields in one pass.
user> (->> [{:x 1 :y 10} {:x 2} {:x 3} {:y 15}]
     (calculate {:x-result (stats :x b/sum b/mean) ;;
                 :y-result (stats :y b/mean)})) ;; <-- Here we're computing mean over :y also.
{:all {:x-result {:count 3, :mean 2.0, :sum 6},
       :y-result {:count 2, :mean 12.5, :sum 25}}}
```

"Extraction" functions can really perform arbitrary operations on
their inputs:

```clojure
user> (->> [{:x 1 :y 10} {:x 2} {:x 3} {:y 15}] 
     (calculate 
       {:x-result (stats :x sum mean) 
        :y-result (stats :y mean) 
        ;; Now accumulate the mean not of a field, but of a function we provide.
        :both (stats #(+ (or (:x %) 0) (or (:y %) 0)) mean)})) 
{:all {:x-result {:mean 2.0, :count 3, :sum 6},
       :y-result {:mean 12.5, :sum 25, :count 2},
       :both {:mean 7.75, :sum 31, :count 4}}}
```

Extraction functions will only be called once per item in the input seq.

Sometimes we will want to compute measures for multiple functions of
the input *together*. However, in the call `(stats extractor-fn
measure1 measure2 ...)`, all the measure functions receive the result
of calling `extractor-fn` on an input item. While we could make the
extractor function return sequences or maps, that would require
modifying all the measure functions to further extract the right
elements. To get around this, `babbage.core` exports three functions
for making the whole item available within a `stats` call: `by`,
`map-with-key`, and `map-with-value`. The interface to all these
functions is similar to that for `stats`: the first argument is an
extractor function, the second argument is the key to be used in the
results map, and the remaining arguments are arbitrarily many measure
functions.

Suppose your input is maps of the form `{:sale x, :user_id y :ts t}`,
representing the amount x of a sale to user `y` at time `t`. You want
to know the total and mean of the sales, and you also want to know how
many individual users made purchases:

```clojure
user> (calculate (stats :sale b/mean b/sum (by :user_id :users b/count-unique))
                 [{:sale 10 :user_id 1} {:sale 20 :user_id 4}
                  {:sale 15 :user_id 1} {:sale 13 :user_id 3}
                  {:sale 25 :user_id 1}])
{:all {:mean 16.6, 
       :users {:unique 3, :count-binned {1 3, 3 1, 4 1}}, 
       :sum 83,
       :count 5}}
```

Or, you might want to know the means and total for each user's
purchases:

```clojure
user> (calculate (stats :sale b/mean b/sum (map-with-key :user_id :user->sales b/mean b/sum))
                 [{:sale 10 :user_id 1} {:sale 20 :user_id 4}
                  {:sale 15 :user_id 1} {:sale 13 :user_id 3}
                  {:sale 25 :user_id 1}])
{:all {:mean 16.6, 
       :user->sales {3 {:mean 13.0, :count 1, :sum 13}, 
                     4 {:mean 20.0, :count 1, :sum 20}, 
                     1 {:mean 16.666666666666668, :count
                     3, :sum 50}}, 
       :sum 83, :count 5}}
```

## Multiple subsets

Simply compute the same measures across multiple subsets, in one pass.

`sets` takes a single argument, a map whose keys are names of sets and
whose values are predicates indicating whether a member of the input
seq belongs in the set:

```clojure
;; We take the previous example, and compute the same measures, but 
;; considering different subsets of elements.
user> (->> [{:x 1 :y 10} {:x 2} {:x 3} {:y 15}] 
     (calculate 
       (sets {:has-y #(-> % :y)}) ;; <-- Compute measures over just those 
                                  ;; elements that have y (in addition 
                                  ;; to all elements).
         {:x-result (stats :x b/sum b/mean) 
          :y-result (stats :y b/mean) 
          :both (stats #(+ (or (:x %) 0) (or (:y %) 0)) b/mean)}))
{:all   {:x-result {:mean 2.0, :count 3, :sum 6}, 
         :y-result {:mean 12.5, :sum 25, :count 2}, 
         :both {:mean 7.75, :sum 31, :count 4}}, 
 :has-y {:x-result {:mean 1.0, :count 1, :sum 1}, 
         :y-result {:mean 12.5, :sum 25, :count 2}, 
         :both {:mean 13.0, :sum 26, :count 2}}}
```

Given an initial predicate map, arbitrary complements, intersections,
and unions can be taken:

```clojure
;; More complex set operations:
user> (def my-sets (-> (sets {:has-y :y
                           :good :good?})
                    ;; add a set called :not-good
                    (complement :good)
                    ;; calculate the intersections of :has-y and :good, and :has-y and :not-good.
                    (intersections :has-y
                                   [:good :not-good])))                                   
#'user/my-sets
user> (def my-fields {:x-result (stats :x b/sum b/mean)
                      :y-result (stats :y b/mean)
                      :both (stats #(+ (or (:x %) 0) (or (:y %) 0)) b/mean)})
#'user/my-fields
user> (calculate my-sets my-fields [{:x 1 :good? true :y 4}
                                    {:x 4 :good? false}
                                    {:x 7 :good? true}
                                    {:x 10 :good? false :y 6}])
{:Shas-y-and-not-goodZ {:x-result {:mean 10.0, :count 1, :sum 10},
                        :y-result {:mean 6.0, :sum 6, :count 1},
                        :both {:mean 16.0, :sum 16, :count 1}},
 :Shas-y-and-goodZ     {:x-result {:mean 1.0, :count 1, :sum 1},
                        :y-result {:mean 4.0, :sum 4, :count 1},
                        :both {:mean 5.0, :sum 5, :count 1}},
 :good                 {:x-result {:mean 4.0, :count 2, :sum 8},
                        :y-result {:mean 4.0, :sum 4, :count 1},
                        :both {:mean 6.0, :sum 12, :count 2}},
 :has-y                {:x-result {:mean 5.5, :count 2, :sum 11},
                        :y-result {:mean 5.0, :sum 10, :count 2},
                        :both {:mean 10.5, :sum 21, :count 2}},
 :all                  {:x-result {:mean 5.5, :count 4, :sum 22},
                        :y-result {:mean 5.0, :sum 10, :count 2},
                        :both {:mean 8.0, :sum 32, :count 4}},
 :not-good             {:x-result {:mean 7.0, :count 2, :sum 14},
                        :y-result {:mean 6.0, :sum 6, :count 1},
                        :both {:mean 10.0, :sum 20, :count 2}}}
```

Predicate functions will only be called once per item in the input seq.


## Efficient computation of inputs

When calculating nontrivial measures over real data, we will often
want to do some transformation of the data to make it tractable, or at
least to minimize verbosity and redundancy in the extraction or
set-definition functions. `babbage.graph` contains tools for
declaring dependency relations among computations and running such
computations efficiently, in the spirit of the
[Prismatic Graph library](http://blog.getprismatic.com/blog/2012/10/1/prismatics-graph-at-strange-loop.html)
and the [Flow library](https://github.com/stuartsierra/flow):

```clojure
user> (defgraphfn sum [xs]
        (apply + xs))
#'user/sum
user> (defgraphfn sum-squared [xs]
        (sum (map #(* % %) xs)))
#'user/sum-squared
user> (defgraphfn count-input :count [xs]
        (count xs))
#'user/count
user> (defgraphfn mean [count sum]
        (double (/ sum count)))
#'user/mean
user> (defgraphfn mean2 [count sum-squared]
        (double (/ sum-squared count)))
#'user/mean2
user> (defgraphfn variance [mean mean2]
        (- mean2 (* mean mean)))
#'user/variance
user> (run-graph {:xs [1 2 3 4]} sum variance sum-squared count-input mean mean2)
{:sum 10
 :count 4
 :sum-squared 30
 :mean 2.5
 :variance 1.25
 :mean2 7.5
 :xs [1 2 3 4]}
```

Functions defined with `defgraphfn` can be invoked like ordinary
Clojure functions defined with `defn` or `fn`, because that's what
they are; the only difference is that the functions carry some
metadata about their dependencies and their names around with them.
Because of this, they can be wrapped by arbitrary other functions, as
long as the metadata is appropriately transferred. (See, for example,
`track-calls` in `babbage.test.graph`.)

When called with `run-graph` (or its variations), the graph function
`mean` can refer to the results of the graph function `sum` simply by
including `sum` in its argument list.

Since (a) we might want to choose dynamically between multiple
functions that can provide the same output to further functions, and
(b) a name that describes a function's output might be undesirable for
the name of the function itself (because it might shadow a core
function, for instance), we can explicitly declare what name a graph
function's output should have when other graph functions want to refer
to it, as in `count-input` in the example above.

`run-graph` takes a map of initial input names and values and any
number of graph functions. It analyzes the dependency graph, throwing
an error in case of unavailable or circular dependencies, and
otherwise runs the functions provided with the input provided. By
default it runs its argument functions in parallel when possible, and
evaluates the results strictly. `run-graph-strategy` can be used to
force sequential evaluation, or lazy evaluation:

```clojure
user> (defgraphfn foo [a]
        (println "in foo"))
#'user/foo
user> (defgraphfn foo [a]
        (println "in foo")
        (inc a))
#'user/foo
user> (defgraphfn bar [foo]
        (println "in bar")
        (* foo foo))
#'user/bar
user> (defgraphfn baz [foo]
        (println "in baz")
        (* foo 2))
#'user/baz
user> (defgraphfn quux [bar baz]
        (println "in quux")
        (- bar baz))
#'user/quux
user> (def _r (run-graph-strategy {:lazy? true} {:a 5} foo bar baz quux))
#'user/_r
user> (:baz _r)
in foo
in baz
12
user> (:baz _r)
12
user> (:quux _r)
in bar
in quux
24
```

`compile-graph` and `compile-graph-strategy` can be used to analyze
the dependency graph once at run time, returning a function that can
be called directly:

```clojure
user> (def f (compile-graph foo bar baz quux))
#'user/f
user> (f {:a 5})
in foo
in bar
in baz
in quux
{:quux 24, :baz 12, :bar 36, :foo 6, :a 5}
user> 
```

`run-graph*` and `run-graph-strategy*` can be used to attempt to
analyze the dependency graph at compile time, falling back to their
un-asterisked analogues if that is not possible.


## Measure functions

### Predefined measure functions 

Several measure functions are predefined in `babbage.provided.core`.

Measure functions that take no arguments:
<table>
    <tr>
        <th>Name</th><th>Effect</th>
    </tr>
    <tr>
        <td><code>sum</code>, <code>prod</code>, <code>max</code>, <code>min</code>, <code>count</code></td>
        <td>Compute the sum, product, maximum, minimum, or count of
        values.</td>
    </tr>
    <tr>
        <td><code>list</code>, <code>set</code></td>
        <td>Accumulate values in a list or a set.</td>
    </tr>
    <tr>
        <td><code>mean</code></td>
        <td>Compute the arithmetic mean of values.</td>
    </tr>
    <tr>
        <td><code>count-binned</code></td>
        <td>Count how often different values have occurred (like
        frequencies).</td>
    </tr>
    <tr>
        <td><code>count-unique</code></td>
        <td>Count how many different values have occurred.</td>
    </tr>
</table>

Measure functions that take arguments:
<table>
    <tr><th>Name</th><th>Arguments</th><th>Effect</th></tr>
    <tr>
        <td><code>ratio</code></td>
        <td><code>of</code>, <code>to</code>, <code>ratio-name</code> (optional)</td>
        <td>Compute the ratio of <code>of</code> to <code>to</code>.
        <code>of</code> must be a keyword  naming a value that a
        measure function will place in the result map; to can either
        be a keyword or a number. If <code>ratio-name</code> is
        provided, it must be a keyword; it will be used as the key in
        the result map for the ratio. Otherwise, <code>of</code> and
        <code>to</code> will be used to create a key of the form
        <code>of</code>-to-<code>to</code>: E.g., <code>(ratio :max
        :min)</code> will use the key  <code>:max-to-min</code>.</td>
   </tr>
   <tr>
       <td><code>histogram</code></td>
       <td><code>width</code></td>
       <td>Compute a histogram whose buckets have width <code>width</code>.</td>
   </tr>
</table>
    
### Defining new measure functions

A measure function is built out of an underlying monoid, or a function
that depends on a measure function, and a public function that
associates the measure with a descriptive name into a map.

For simple functions the `defstatfn` macro in `babbage.core` and the
`monoid` function in `babbage.monoid` can be used to reduce
boilerplate. For instance, the following would record the first
non-nil value in the input sequence:

```clojure
user> (def m-fst (monoid (fn [a b] a) nil))
#'user/m-fst
user> (defstatfn fst m-fst)
#'user/fst
user> (calculate (stats :x fst sum) [{:x nil} {:x 2} {:x 3} {:x 4}])
{:all {:sum 9, :fst 2}}
```

The `statfn` macro can be used to create an anonymous measure function
that depends on runtime values: see `babbage.provided.core/histogram`
and `babbage.provided.core/ratio` for examples.

The `monoid` function takes a binary operation and a value that is a
left and right identity when the operation has non-nil arguments (nil
is special-cased to always be an identity) and returns a function that
creates instances of `babbage.monoid/Monoid`. More complex measures
can implement the protocol directly; see `babbage.monoid.gaussian` for
an example.

Measures that depend on already-computed measures (e.g. the ratio of
one to another) are also a combination of an underlying function that
does the actual computation and a public function that associates the
measure with a name in a map. In this case, both functions declare
their dependencies:

```clojure
;; Keeping track of unique counts is dependent on keeping track of the seen ones.
;; "defnmeta" just associates the attr-map metadata with both the var and the function
(defnmeta d-count-unique {:requires #{:count-binned}} [m]
  (count (get m :count-binned)))
  
(defstatfn count-unique d-count-unique :requires count-binned :name :unique)
```

`d-count-unique` declares that it expects the result map it is passed
to contain the `:count-binned` key. `count-unique` uses
`d-count-unique` to compute its results, says that its result is named
`:unique`, and requires the *function* `count-binned`. Using
`count-unique` will result in `count-binned` being used as well. In
this case, there is some redundancy in the declarations, because
`count-unique`'s requirements are known statically.

## License

Copyright (c) ReadyForZero, Inc. All rights reserved.

The use and distribution terms for this software are covered by the Eclipse Public License 1.0 (http://opensource.org/licenses/eclipse-1.0.php)

## Contributors

[Ben Wolfson](http://github.com/bwo) (wolfson@readyforzero.com)

<hr>

<small>
At each increase of knowledge, as well as on the contrivance of every new tool, human labour becomes abridged. -- Charles Babbage
</small>
