+++
Categories = ["Cxx"]
Description = "Selection Algorithms for Graph Partitioning"
Tags = ["Cxx"]
date = "2015-12-13T13:53:51+01:00"
title = "Selection Algorithms for Graph Partitioning"

+++

In which I optimize a graph partitioner by carefully extracting the algorithm's core requirements and then selecting appropriate C++14 Stdlib algorithms.

## Motivation

The following paper introduces a simple yet powerful graph partitioning technique called Inertial Flow.

> [On Balanced Separators in Road Networks](http://sommer.jp/roadseparator.htm) (doi: `10.1007/978-3-319-20086`)

The basic idea is this:

- Sort vertices "spatially" by a linear combination of their coordinate's latitude and longitude
- Take the first `k` nodes forming the sources and the last `k` nodes forming the sinks
- Run a single Max-Flow algorithm from sources to sinks and return the corresponding Min-Cut

Setting the balance parameter `k` e.g. to `0.25 * |V|` guarantees for a balanced partition, since the disjoint sets have at least `k` vertices.
As an optimization you can try different spatial orders, that is visually you rotate a line `n` times, run the algorithm and return the best cut.
For partitioning your graph you then recurse on both disjoint vertex sets, until you reach a certain depth or a minimum number of nodes.
That's it. Really, it's that simple and it works surprisingly well!

Take a look at the following map I generated from dumping the partitioner's graph using [tippecanoe](https://github.com/mapbox/tippecanoe) to build  a simplified vector tilesets.

<iframe width='100%' height='500px' frameBorder='0' src='https://api.mapbox.com/styles/v1/danieljh/cii4lhkpn005wb8lzwjj43ipy.html?title=true&access_token=pk.eyJ1IjoiZGFuaWVsamgiLCJhIjoiTnNYb25JSSJ9.vYOnsuu1zeKcGW2nj0uJZw#9.658735200693558/52.4992304153767/13.449950544246803/0'></iframe>

This is a single algorithm run on Berlin with `k = 0.25 * |V|` and a simple spatial order by longitude. The blue points represent the first `k` vertices under that spatial order forming the source.
The red points represent the last `k` vertices under that spatial order forming the sinks.
Running a single Max-Flow algorithms such as Edmonds–Karp, Push–Relabel or Dinic's algorithm from sources to sinks results in the corresponding Min-Cut that is represented by the black points.

## Deriving the Spatial Order

The spatial order is derived by a binary function `spatially` of two vertices, that compares a linear combination of their coordinate's latitude and longitude.

The first and last `k` vertices can then be determined by using [`std::sort`](http://en.cppreference.com/w/cpp/algorithm/sort) as described in the paper.

    sort(begin(vertices), end(vertices), spatially);

We can then take the first and last `k` vertices forming sources and sinks, respectively.
But wait, we can do better: there is no need to sort the vertices in the "middle". Let's to less work by using [`std::partial_sort`](http://en.cppreference.com/w/cpp/algorithm/partial_sort).

    partial_sort(begin(vertices), begin(vertices) + k, end(vertices), spatially);
    partial_sort(rbegin(vertices), rbegin(vertices) + k, rend(vertices), flip(spatially));

This sorts the first `k` vertices, and then sorts the last `k` vertices with flipped arguments for the binary function (we flip the arguments instead of `std::not2` so that the relation is still irreflexive). Great, less work and good enough for our use-case!

But do we really need a complete ordering of the first and last `k` vertices? After all we only need the order to determine sources and sinks for the Max-Flow algorithm.
We are neither interested in which order the first `k` are, nor in which order the last `k` elements are.
Take a look at the visualization: all what matters is the vertex property "first `k` in the spatial order" (blue) or "last `k` in the spatial order" (red). It does not matter how the red or blue points are ordered in the set of red and blue vertices, respectively.

Let's give this another try. With [`std::nth_element`](http://en.cppreference.com/w/cpp/algorithm/nth_element) we can get the element at position `k` that would occur there if the full range was sorted.
In addition, all the elements before the `k`th element are "less or equal" to that element. Interesting, so this is a variation of insertion sort.

    nth_element(begin(vertices), begin(vertices) + k, end(vertices), spatially);
    nth_element(rbegin(vertices), rbegin(vertices) + k, rend(vertices), flip(spatially));

Visually speaking, this tells us "these are red", and "these are blue", without any ordering guarantees in the sets.

It is crucial to understand the difference to partial sorting. Suppose we have a range of integers.

    8 7 6 4 5 3 3 2

Using [`std::partial_sort`](http://en.cppreference.com/w/cpp/algorithm/partial_sort) on the first and last three elements results in the following.

    2 3 3 _ _ 6 7 8

In contrast, using [`std::nth_element`](http://en.cppreference.com/w/cpp/algorithm/nth_element) on position three from the beginning and end gives you the following guarantees:

    _ _ 3 _ _ 6 _ _
    ____|     |____
    <= 3       >= 6

The subranges are no longer sorted, but satisfy the binary function with respect to the selected element. This allows the algorithm to do less work then the partial or even the full sort algorithm.

Now that we know how [`std::nth_element`](http://en.cppreference.com/w/cpp/algorithm/nth_element) works and what guarantees it gives us, we can even go further: the second [`std::nth_element`](http://en.cppreference.com/w/cpp/algorithm/nth_element) does not have to take a look at the full range, since we know that we already reordered the first `k` elements with the flipped binary function. We therefore come up with the following.

    nth_element(begin(vertices), begin(vertices) + k, end(vertices), spatially);
    nth_element(rbegin(vertices), rbegin(vertices) + k, rend(vertices) - k, flip(spatially));

This reorders the first `k` elements by looking at the full range and then reorders the last `k` elements by only looking at the `k + (size - k)` elements from the end.


## Discussion

I talked to Christian Sommer, one of the paper's authors, about this. He acknowledged there is no need for a full ordering that [`std::sort`](http://en.cppreference.com/w/cpp/algorithm/sort) gives you as described in the paper.
Furthermore he argued that you could fully sort your `n` spatial orders and keep them around for all recursion steps, which would require more memory and algorithms that can select the subgraph's vertices from the orders.

As of writing this, the prototype partitioner still uses the Edmonds-Karp algorithms.
We can probably gain significant improvements by using Dinic's algorithm, shadowing the small improvements achieved here.
This does not mean that we should not optimize for easy wins as it was with this case; after all the final reordering optimization on its own is faster than a full sort by a factor of 4-7 from what I saw in a few experiments.
Also, this is where the fun is in engineering and programming :-)


## Further Reading

- Sean Parent [has a few papers and presentations](https://github.com/sean-parent/sean-parent.github.io/wiki/Papers-and-Presentations) in which he explains similar clever Stdlib algorithm usage 
- Alexander Stepanov's "From Mathematics to Generic Programming" and "Elements of Programming" go in great detail about algorithm requirements and guarantees


## Summary

Inertial Flow is a simple yet powerful graph partitioning technique that requires a spatial order.
Deriving the spatial order can be optimized by carefully looking at the algorithm's requirements.
Know your Stdlib, in particular be familiar with more "exotic" algorithms such as [`std::nth_element`](http://en.cppreference.com/w/cpp/algorithm/nth_element) and [`std::rotate`](http://en.cppreference.com/w/cpp/algorithm/rotate) --- or of course equivalent algorithms in your language of choice.
