# Introduction

Several tables in the OpenType format are formed internally by a graph of subtables. Parent node's
reference their children through the use of positive offsets, which are typically 16 bits wide.
Since offsets are always positive this forms a directed acyclic graph. For storage in the font file
the graph must be given a topological ordering and then the subtables packed in serial according to
that ordering. Since 16 bit offsets have a maximum value of 65,535 if the distance between a parent
subtable and a child is more then 65,535 bytes then it's not possible for the offset to encode that
edge.

For many fonts with complex layout rules (such as Arabic) it's not unusual for the tables containing
layout rules ([GSUB/GPOS](https://docs.microsoft.com/en-us/typography/OpenType/spec/gsub)) to be
larger than 65kb. As a result these types of fonts are susceptible to offset overflows when
serializing to the binary font format.

Offset overflows can happen for a variety of reasons and require different strategies to resolve:
*  Simple overflows can often be resolved with a different topological ordering.
*  If a subtable has many parents this can result in the link from furthest parent(s)
   being at risk for overflows. In these cases it's possible to duplicate the shared subtable which
   allows it to be placed closer to it's parent.
*  If subtables exist which are themselves larger than 65kb it's not possible for any offsets to point
   past them. In these cases the subtable can usually be split into two smaller subtables to allow
   for more flexibility in the ordering.
*  In GSUB/GPOS overflows from Lookup subtables can be resolved by changing the Lookup to an extension
   lookup which uses a 32 bit offset instead of 16 bit offset.
   
In general there isn't a simple solution to produce an optimal topological ordering for a given graph.
Finding an ordering which doesn't overflow is a NP hard problem. Existing solutions use heuristics
which attempt a combination of the above strategies to attempt to find a non-overflowing configuration.
   
The HarfBuzz subsetting library
[includes a repacking algorithm](https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-repacker.hh)
which is used to resolve offset overflows that are present in the subsetted tables it produces. This
document provides a deep dive into how the HarfBuzz repacking algorithm works.

Other implementations exist, such as in
[fontTools](https://github.com/fonttools/fonttools/blob/7af43123d49c188fcef4e540fa94796b3b44e858/Lib/fontTools/ttLib/tables/otBase.py#L72), however these are not covered in this document.

# Foundations

There's four key pieces to the HarfBuzz approach:

*  Subtable Graph: a table's internal structure is abstracted out into a lightweight graph
   representation where each subtable is a node and each offset forms an edge. The nodes only need
   to know how many bytes the corresponding subtable occupies. This lightweight representation can
   be easily modified to test new orderings and strategies as the repacking algorithm iterates.

*  [Topological sorting algorithm](https://en.wikipedia.org/wiki/Topological_sorting): an algorithm
   which given a graph gives a linear sorting of the nodes such that all offsets will be positive.
   
*  Overflow check: given a graph and a topological sorting it checks if there will be any overflows
   in any of the offsets. If there are overflows it returns a list of (parent, child) tuples that
   will overflow. Since the graph has information on the size of each subtable it's straightforward
   to calculate the final position of each subtable and then check if any offsets to it will
   overflow.
   
*  Offset resolution strategies: given a particular occurrence of an overflow these strategies
   modify the graph to attempt to resolve the overflow.
   
# High Level Algorithm

```
def repack(graph):
  graph.topological_sort()

  while (overflows = graph.will_overflow()):
    for overflow in overflows:
      apply_offset_resolution_strategy (overflow, graph)
    graph.topological_sort()
```

The actual code for this processing loop can be found [here](https://github.com/harfbuzz/harfbuzz/blob/main/src/hb-repacker.hh#L682).

# Topological Sorting Algorithms

The HarfBuzz repacker uses two different algorithms for topological sorting:
*  [Kahn's Algorithm](https://en.wikipedia.org/wiki/Topological_sorting#Kahn's_algorithm)
*  Sorting by shortest distance

Kahn's algorithm is approximately twice as fast as the shortest distance sort so that is attempted
first (only on the first topological sort). If it fails to eliminate overflows then shortest distance
sort will be used for all subsequent topological sorting operations.
   
## Shortest Distance Sort

This algorithm orders the nodes based on total distance to each node. Nodes with a shorter distance
are ordered first.

The "weight" of an edge is the sum of the size of the sub-table being pointed to plus 2^16 for a 16 bit
offset and 2^32 for a 32 bit offset.

The distance of a node is the sum of all weights along the shortest path from the root to that node
plus a priority modifier (used to change where nodes are placed by moving increasing or
decreasing the effective distance). Ties between nodes with the same distance are broken based
on the order of the offset in the sub table bytes.

The shortest distance to each node is determined using
[Djikstra's algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm). Then the topological
ordering is produce by applying a modified version of Kahn's algorithm that uses a priority queue
based on the shortest distance to each node.

## Optimizing the Sorting

The topological sorting operation is the core of the repacker and is run on each iteration so it needs
to be as fast as possible. There's a few things that are done to speed up subsequent sorting
operations:

*  The number of incoming edges to each node is cached. This is required by the Kahn's algorithm
   portion of both sorts. Where possible when the graph is modified we manually update the cached
   edge counts of affected nodes.
   
*  The distance to each node is cached. Where possible when the graph is modified we manually update
   the cached distances of any affected nodes.

Caching these values allows the repacker to avoid recalculating them for the full graph on each
iteration.

The other important factor to speed is a fast priority queue which is a core data structure to
the topological sorting algorithm. Currently a basic heap based queue is used. Heap based queue's
don't support fast priority decreases, but that can be worked around by just adding redundant entries
to the priority queue and filtering the older ones out when popping off entries. This is based
on the recommendations in
[a study of the practical performance of priority queues in Dijkstra's algorithm](https://www3.cs.stonybrook.edu/~rezaul/papers/TR-07-54.pdf)

# Offset Resolution Strategies

For each overflow in each iteration the algorithm will attempt to apply offset overflow resolution
strategies to eliminate the overflow. The type of strategy applied is dependent on the characteristics
of the overflowing link:

*  If the overflowing offset is pointing to a subtable with more than one incoming edge: duplicate
   the node so that the overflowing offset is pointing at it's own copy of that node.
   
*  Otherwise, attempt to move the child subtable closer to it's parent. This is accomplished by
   raising the priority of all children of the parent. Next time the topological sort is run the
   children will be ordered closer to the parent.
   
# Test Cases

The HarfBuzz repacker has tests defined using generic graphs: https://github.com/harfbuzz/harfbuzz/blob/main/src/test-repacker.cc
   
# Future Improvements

The above resolution strategies are not sufficient to resolve all overflows. For example consider
the case where a single subtable is 65k and the graph structure requires an offset to point over it.

The current HarfBuzz implementation is suitable for the vast majority of subsetting related overflows.
Subsetting related overflows are typically easy to solve since all subsets are derived from a font
that was originally overflow free. A more general purpose version of the algorithm suitable for font
creation purposes will likely need some additional offset resolution strategies:

*  Currently only children nodes are moved to resolve offsets. However, in many cases moving a parent
   node closer to it's children will have less impact on the size of other offsets. Thus the algorithm
   should use a heuristic (based on parent and child subtable sizes) to decide if the children's
   priority should be increased or the parent's priority decreased.
   
*  Many subtables can be split into two smaller subtables without impacting the overall functionality.
   This should be done when an overflow is the result of a very large table which can't be moved
   to avoid offsets pointing over it.
   
*  Lookup subtables in GSUB/GPOS can be upgraded to extension lookups which uses a 32 bit offset.
   Overflows from a Lookup subtable to it's child should be resolved by converting to an extension
   lookup.
   
Once additional resolution strategies are added to the algorithm it's likely that we'll need to
switch to using a [backtracking algorithm](https://en.wikipedia.org/wiki/Backtracking) to explore
the various combinations of resolution strategies until a non-overflowing combination is found. This
will require the ability to restore the graph to an earlier state. It's likely that using a stack
of undoable resolution commands could be used to accomplish this.
