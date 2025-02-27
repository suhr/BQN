*View this file with results and syntax highlighting [here](https://mlochbaum.github.io/BQN/implementation/primitive/sort.html).*

# Implementation of ordering functions

The [ordering functions](../../doc/order.md) are Sort (`∧∨`), Grade (`⍋⍒`), and Bins (`⍋⍒`). Although these are well-studied—particularly sorting, and then binary search or "predecessor search"—there are many recent developments, as well as techniques that I have not found in the literature. The three functions are closely related but have important differences in what algorithms are viable. Sorting is a remarkably deep problem with different algorithms able to do a wide range of amazing things, and sophisticated ways to combine those. It is by no means solved. In comparison, Bins is pretty tame.

There's a large divide between ordering compound data and simple data. For compound data comparisons are expensive, and the best algorithm will generally be the one that uses the fewest comparisons. For simple data they fall somewhere between cheap and extremely cheap, and fancy branchless and vectorized algorithms are the best.

## On quicksort versus merge sort

Merge sort is better. It is deterministic, stable, and has optimal worst-case performance. Its pattern handling is better: while merge sort handles "horizontal" patterns and quicksort does "vertical" ones, merge sort gets useful work out of *any* sequence of runs but in-place quicksort will quickly mangle its analogue until it may as well be random.

But that doesn't mean merge sort is always faster. Quicksort seems to work a little better branchlessly. For sorting, quicksort's partitioning can reduce the range of the data enough to use an extremely quick counting sort. Partitioning is also a natural fit for binary search, where it's mandatory for sensible cache behavior with large enough arguments. So it can be useful. But it doesn't merge, and can't easily be made to merge, and that's a shame.

The same applies to the general categories of partitioning sorts (quicksort, radix sort, samplesort) and merging sorts (mergesort, timsort, multimerges). Radix sorts are definitely the best for some types and lengths, although the scattered accesses make their performance unpredictable and I think overall they're not worth it. A million uniformly random 4-byte integers is nearly the best possible case for radix sort, so the fact that this seems to be the go-to sorting benchmark means radix sorting looks better than it is.

## On binary search

Binary searches are very easy to get wrong. Do not write `(hi+lo)/2`: it's not safe from overflows. I always follow the pattern given in the first code block [here](https://pvk.ca/Blog/2015/11/29/retrospective-on-binary-search-and-on-compression-slash-compilation/). This code will never access the value `*base`, so it should be considered a search on the `n-1` values beginning at `base+1` (the perfect case is when the number of values is one less than a power of two, which is in fact how it has to go). It's branchless and always takes the same number of iterations. To get a version that stops when the answer is known, subtract `n%2` from `n` in the case that `*mid < x`.

## Compound data

Array comparisons are expensive. The goal here is almost entirely to minimize the number of comparisons. Which is a much less complex goal than to get the most out of modern hardware, so the algorithms here are simpler.

For **Sort** and **Grade**, use Timsort. It's time-tested and shows no signs of weakness (but do be sure to pick up a fix for the bug discovered in 2015 in formal verification). Hardly different from optimal comparison numbers on random data, and outstanding pattern handling. Grade can be done either by selecting from the original array to order indices or by moving the data around in the same order as the indices. I think the second of these ends up being substantially better for small-ish elements.

For **Bins**, use a branching binary search: see [On binary search](#on-binary-search) above. But there are also interesting (although, I expect, rare) cases where only one argument is compound. Elements of this argument should be reduced to fit the type of the other argument, then compared to multiple elements. For the right argument, this just means reducing before doing whatever binary search is appropriate to the left argument. If the left argument is compound, its elements should be used as partitions. Then switch back to binary search only when the partitions get very small—probably one element.

## Simple data

The name of the game here is "branchless".

Sorting algorithms of interest are counting sort and [pdqsort](https://github.com/orlp/pdqsort) (some improvements of my own to be described here later). However, these are both unusable for Grade.

For small-range Grade, counting sort must be replaced with bucket sort, at a significant performance cost. I don't have any method I'm happy with for other data. Stabilizing pdqsort by sorting the indices at the end is possible but slow. [Wolfsort](https://github.com/scandum/wolfsort) is a hybrid radix/merge sort that might be better.

[IPS⁴o](https://github.com/ips4o/ips4o) is a horrifyingly complicated samplesort thing. Stable, I think. For very large arrays it probably has the best memory access patterns, so a few samplesort passes could be useful.

A branchless binary search is adequate for Bins but in many cases—very small or large `𝕨`, and small range—there are better methods.

### Counting and bucket sort

Both counting and bucket sort are small-range algorithms that begin by counting the number of each possible value. Bucket sort, as used here, means that the counts are then used to place values in the appropriate position in the result in another pass. Counting sort does not read from the initial values again and instead reconstructs them from the counts. It might be written `(/≠¨∘⊔)⌾(-⟜min)` in BQN, with `≠¨∘⊔` as a single efficient operation.

Bucket sort can be used for Grade or sort-by (`⍋⊸⊏`), but counting sort only works for sorting itself. It's not-even-unstable: there's no connection between result values and the input values except that they are constructed to be equal. But with [fast Indices](replicate.md#non-booleans-to-indices), Counting sort is vastly more powerful, and is effective with a range four to eight times the argument length. This is large enough that it might pose a memory usage problem, but the memory use can be made arbitrarily low by partitioning.

### Partitioning

In-place quicksort relies on a partitioning algorithm that exchanges elements in order to split them into two contiguous groups. The [Hoare partition scheme](https://en.wikipedia.org/wiki/Quicksort#Hoare_partition_scheme) does this, and [BlockQuicksort](https://github.com/weissan/BlockQuicksort) showed that it can be performed quickly with branchless index generation; this method was then adopted by pdqsort. But the [bit booleans to indices](replicate.md#booleans-to-indices) method is faster and fits well with vectorized comparisons.

It's simplest to define an operation `P` that partitions a list `𝕩` according to a boolean list `𝕨`. Partitioning permutes `𝕩` so that all elements corresponding to 0 in `𝕨` come before those corresponding to 1. The quicksort partition step, with pivot `t`, is `(t≤𝕩)P𝕩`, and the comparison can be vectorized. Interleaving comparison and partitioning in chunks would save memory (a fraction of the size of `𝕩`, which should have 32- or 64-bit elements because plain counting sort is best for smaller ones) but hardly speeds things up: only a few percent, and only for huge lists with hundreds of millions of elements. The single-step `P` is also good for Bins, where the boolean `𝕨` will have to be saved.

For binary search `𝕨⍋𝕩`, partitioning allows one pivot element `t` from `𝕨` to be compared to all of `𝕩` at once, instead of the normal strategy of working with one element from `𝕩` at a time. `𝕩` is partitioned according to `t≤𝕩`, then result values are found by searching the first half of `𝕨` for the smaller elements and the second half for the larger ones, and then they are put back in the correct positions by reversing the partitioning. Because Hoare partitioning works by swapping independent pairs of elements, `P` is a self inverse, identical to `P⁼`. So the last step is simple, provided the partitioning information `t≤𝕩` is saved.

### Binary search

Reminder that we're talking about simple, not [compound](#compound-data) data. The most important thing is just to have a good branchless binary search (see [above](#on-binary-search)), but there are other possible optimizations.

If `𝕨` is extremely small, use a vector binary search as described in "Sub-nanosecond Searches" ([video](https://dyalog.tv/Dyalog18/?v=paxIkKBzqBU), [slides](https://www.dyalog.com/user-meetings/uploads/conference/dyalog18/presentations/D08_Searches_Using_Vector_Instructions.zip)). For 1-byte elements there's also a vectorized method that works whenever `𝕨` has no duplicates: create two lookup tables that go from multiples of 8 (5-bit values, after shifting) to bytes. One is a bitmask of `𝕨`, so that a lookup gives 8 bits indicating which possible choices of the remaining 3 bits are in `𝕨`. The other gives the number of values in `𝕨` less than the multiple of 8. To find the result of Bins, look up these two bytes. Mask off the bitmask to include only bits for values less than the target, and sum it (each of these steps can be done with another lookup, or other methods depending on instruction set). The result is the sum of these two counts.

It's cheap and sometimes worthwhile to trim `𝕨` down to the range of `𝕩`. After finding the range of `𝕩`, binary cut `𝕨` to a smaller list that contains the range. Stop when the middle element fits inside the range, and search each half of `𝕨` for the appropriate endpoint of the range.

If `𝕩` is small-range, then a lookup table method is possible. Check the length of `𝕨` because if it's too large then this method is slower! The approach is simply to create a table of the number of elements in `𝕨` with each value, then take a prefix sum. In BQN, ``𝕩⊏+`∾≠¨⊔𝕨``, assuming a minimum of 0.

[Partitioning](#partitioning) allows one pivot `t` from `𝕨` to be compared with all of `𝕩` at once. Although the comparison `t≤𝕩` can be vectorized, the overhead of partitioning still makes this method a little slower per-comparison than sequential binary search *when* `𝕨` *fits in L1 cache*. For larger `𝕨` (and randomly positioned `𝕩`) cache churn is a huge cost and partitioning can be many times faster. It should be performed recursively, switching to sequential binary search when `𝕨` is small enough. Unlike quicksort there is no difficulty in pivot selection: always take it from the middle of `𝕨` as in a normal binary search. However, there is a potential issue with memory. If `𝕩` is unbalanced with respect to `𝕨`, then the larger part can be nearly the whole length of `𝕩` (if it's all of `𝕩` partitioning isn't actually needed and it doesn't need to be saved). This can require close to `2⋆⁼≠𝕨` saved partitions of length `≠𝕩`, while the expected use would be a total length `≠𝕩`.
