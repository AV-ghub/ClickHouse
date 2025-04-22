## [RoaringBitmap](https://github.com/RoaringBitmap/RoaringBitmap)
### Introduction
Bitsets, also called bitmaps, are commonly used as fast data structures. Unfortunately, they can use too much memory. To compensate, we often use compressed bitmaps.

Roaring bitmaps are compressed bitmaps which tend to outperform conventional compressed bitmaps such as WAH, EWAH or Concise. In some instances, roaring bitmaps can be hundreds of times faster and they often offer significantly better compression. They can even be faster than uncompressed bitmaps.

### When should you use a bitmap?
Sets are a fundamental abstraction in software. They can be implemented in various ways, as hash sets, as trees, and so forth. In databases and search engines, sets are often an integral part of indexes. For example, we may need to maintain a set of all documents or rows (represented by numerical identifier) that satisfy some property. Besides adding or removing elements from the set, we need fast functions to compute the intersection, the union, the difference between sets, and so on.

When the bitset approach is applicable, it can be orders of magnitude faster than other possible implementation of a set (e.g., as a hash set) while using several times less memory.

There is a big problem with these formats however that can hurt you badly in some cases: **there is no random access**. If you want to check whether a given value is present in the set, you have to start from the beginning and "uncompress" the whole thing. This means that **if you want to intersect a big set with a large set, you still have to uncompress the whole big set** in the worst case...

**Roaring** solves this problem. It works in the following manner. It **divides the data into chunks of 216 integers** (e.g., [0, 216), [216, 2 x 216), ...). **Within a chunk, it can use an uncompressed bitmap**, a simple list of integers, or a list of runs. Whatever format it uses, they all allow you to check for the presence of any one value quickly (e.g., with a binary search). The net result is that Roaring can compute many operations much faster than run-length-encoded formats like WAH, EWAH, Concise... Maybe surprisingly, Roaring also generally **offers better compression** ratios.
