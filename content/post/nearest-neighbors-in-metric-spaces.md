+++
Categories = []
Description = ""
Tags = []
date = "2017-04-30T14:39:33+02:00"
title = "Nearest Neighbor Queries in Metric Spaces"

+++

[R-trees](https://medium.com/@agafonkin/a-dive-into-spatial-search-algorithms-ebd0c5e39d2a) can be used for fast spatial nearest neighbor queries in euclidean spaces.
But what about non-euclidean spaces — e.g. efficiently querying for similar words, time series, images or videos?
In the following I outline my journey towards indices for metric spaces with literature highlights going back to the 70s.


## Motivation

Here's an interesting problem I stumbled upon lately

* You have a database with `n` integers
* You want to efficiently find all `k` "nearest" integers in your database
* "near" is defined in terms of number of bits the integers differ by, the [Hamming distance](https://en.wikipedia.org/wiki/Hamming_distance)

With 8 bit numbers your database could look like the following

```
Integers
--------
00000001
01000010
00100001
10000000
01010101
10101010
```

for a query `db.nearest(01000000, 2)` you want to find all integers in the database with a Hamming distance of at most `d=2` _to the integer `01000000`_:

```
Integers | Distance
---------|---------
00000001 |        2
01000010 |        1
00100001 |        3
10000000 |        2
01010101 |        3
10101010 |        5
```

The question now is how can you pre-process your database such that _any_ `nearest` query can be answered efficiently.
Doing a linear scan over all integers in the database and computing all Hamming distances to the query integer is clearly not scalable.
An interesting idea is to pre-compute all possible integers with a Hamming distance of at most `d` to the query integer.
At query time any fast lookup scheme can then be used to check if an integer is in the database.
The problem here is the combinatorial explosion as soon as your integers are wider than 8 bits or `d` gets large.

As an example, with 32 bit wide integers and allowing for 3 bits to differ there are already 5489 possible integers you have to pre-compute and do lookups for:

```python
def choose(n, k):
    return math.factorial(n) // (math.factorial(k) * math.factorial(n - k))

sum([choose(32, b) for b in range(0, 4)])
5489
```

Not scalable for the generic use-case.

## Metric Trees

Enter metric spaces.

[Metrics](https://en.wikipedia.org/wiki/Metric_\(mathematics\)) abstract over the notion of distances between two items in a set, given only the following properties on the distance function:

* distance(x, y) >= 0  (`non-negative`)
* distance(x, y) == 0 <=> x == y  (`zero distance iff. identity`)
* distance(x, y) == distance(y, x)  (`symmetry`)
* distance(x, z) <= distance(x, y) + distance(y, z)  (`triangle inequality`)

For each property you can check it holds for the [Hamming distance](https://en.wikipedia.org/wiki/Hamming_distance) — and in fact the Hamming distance is a metric for bit vectors.

Back to the 70s.
Burkhard and Keller just proposed datastructures for fast nearest neighbor queries in metric spaces.

> ["Some approaches to best-match file searching", Burkhard and Keller](http://dl.acm.org/citation.cfm?id=362025)

The Burkhard-Keller metric tree (BK-tree) is what allows us to do fast nearest neighbor queries in metric spaces.
Nodes in the BK-tree consist of a single item and arbitrary sub-trees with distance labels on its edges.
To construct a BK-tree you take an arbitrary item in your database and set it as the root node.
Let's use `00000001` as root node in the example.

Inserting items happens one at a time:
traverse the tree and follow edges labeled `distance(node, item)`.
If there is no such edge insert a new child.
Otherwise recurse down the tree.
In the example with root node `00000001` you now insert `01000010` from the database next:
the Hamming distance between `00000001` and `01000010` is 3 and the root nodes does not have an edge with label 3 yet.
Therefore, you add an edge with label 3 to the root and a node with `01000010` as the edge's target.

```



           3   01000010
          /
00000001




```

You continue to do the same for all items in the database:

```

           5   10101010
         /
        |  3   01000010
        | /
00000001
        | \
        |  2   10000000
         \
           1   00100001 
```

Until you want to insert `01010101` which results in a Hamming distance of 3 to the root node.
There is already an edge labeled 3.
Therefore, you recursively descend into the sub-tree with the edge label 3 and start from scratch:
compute the Hamming distance between the sub-tree's root `01000010` and the item to insert `01010101`, which is 4.
You add an edge with label 4 to the sub-tree's root and a node with `01010101` as the edge's target:

```

           5   10101010
         /
        |  3   01000010  - 4 -  01010101
        | /
00000001
        | \
        |  2   10000000
         \
           1   00100001 
```

The BK-tree structure now has the following property:
all items under a sub-tree with an edge label `l` have the Hamming distance `l` _to the root node_.
This property and the triangle inequality is key for efficient lookups:
querying for nearest neighbors happens by recursively traversing the tree following edges where the edge label is in the range `[dist - d, dist + d]` with `dist` being the Hamming distance between the query item and each node and `d` is the maximum allowed difference.

In the example the query `db.nearest(01000000, 2)` starts out computing the Hamming distance to the tree's root node `00000001` which is 2. We save the root node (since it fits our criteria) and recursively descend into sub-trees with edge labels in the range `[2 - 2, 2 + 2]`.

To give some perspective, here is an example with roughly 100k integers of width 32.
Using the Hamming distance constructing a BK-tree is nearly instantaneous.
And because the tree is neatly balanced (not skewed, not too deep but also not too shallow) lookups are a matter of following a highly restricted set of edges, with the number of distance computations being in the low single digit percentage of the full database's size.

Here is a SVG for my BK-tree limited to the first 50 integers — reason for this being DOT having troubles with layouting and not terminating for the full tree.

[<img src="/images/nearest-neighbors-in-metric-spaces/bk.svg">](/images/nearest-neighbors-in-metric-spaces/bk.svg)

The reason the BK-tree works so well here are discrete distances:
the Hamming distances can only take on integral values in the range `[0, width]` which means each node has at most 32 children and — depending on how your data is distributed — is nicely balanced for efficient lookups.

The BK-tree's lookup is in `O(n)` (not better than a linear scan over the database in the worst case) when the tree is completely skewed.
This can happen in the pathological case where _all pairwise distances_ are the same.
In practice the runtime highly depends on your data and the BK-tree is suited for discrete distances.

[Here](https://gist.github.com/daniel-j-h/8418cd89789c3fe611a8362161d86a6a) is my BK-tree implementation in case you want to play with it.

## Use-Cases

With the BK-tree in your toolbox, let's see how it can be used for similarity queries.

The Hamming distance already allows for similarity queries with a database full of binary feature vectors.
To make this work you one-hot encode features such as "likes trees", "is a cat-person", etc. into a bit vector of constant size, think: `(1, 0, 0, 1, 0, 1, 0, 0)`.
Nearest neighbor queries with the BK-tree will now efficiently answer similarity queries allowing you to search for items with at most `d` differences. Great!

For string similarity the [Levenshtein distance](https://en.wikipedia.org/wiki/Levenshtein_distance) is a metric and the BK-tree can be used, too.
Another approach for string similarity is to use [n-grams](https://en.wikipedia.org/wiki/N-gram) and the [Jaccard distance](https://en.wikipedia.org/wiki/Jaccard_index) between the sets of n-grams.
Or approximate the Jaccard distance between the n-grams using a locality-sensitive hashing scheme such as [MinHash](https://en.wikipedia.org/wiki/MinHash).

Similar approaches work for time series, and even audio and video if you fingerprint the media first and then do similarity searches using metric trees.


## Further Reading

The BK-tree is perfectly suited for well-behaving and discrete distances.
For more advanced use-cases it can make sense to have a look at other datastructures.

The Vantage-Point tree (VP-tree) can be used for fast nearest neighbor queries in metric spaces.

> ["Data Structures and Algorithms for Nearest Neighbor Search in General Metric Spaces", Yianilos](http://dl.acm.org/citation.cfm?id=313789)

Constructing the VP-tree works by picking a vantage point and a distance to it bisecting the space:
half of the items are within the distance "radius", half of the items are outside of the distance "radius", creating a binary tree.
The VP-tree paper has a great visualization on [page 2, Figure 1: VP-tree decomposition](http://pnylab.com/pny/papers/vptree/main.html).

Additional improvements can be made with Geometric Near-neighbor Access Trees (GNATs).

> ["Near Neighbor Search in Large Metric Spaces", Brin](http://dl.acm.org/citation.cfm?id=673006)

The key idea for GNATs is to split `k` far apart split points, recursive Voronoi partitioning

> ["Geometric Near-neighbor Access Tree (GNAT) revisited", Kimmo Fredriksson](https://arxiv.org/abs/1605.05944)

https://en.wikipedia.org/wiki/Locality-sensitive_hashing


## Summary

The BK-tree can efficiently answer nearest neighbor queries for metric spaces and is suited for discrete distances.
For more general use-cases (especially continuous distances) the VP-tree and GNATs are better suited.
