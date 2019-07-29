---
title: "Linear Sort Algorithm"
category: Algorithm
tag: algorithm
---
# Bucket Sort #
 1. Create m empty buckets (Or lists) in order.
 2. Split the data into the m buckets
 3. Sort each bucket internally using for example: [quick sort or merge sort](https://blog.csdn.net/weixin_42909055/article/details/97289674)
 4. Concatenate all sorted buckets

With the assumption that the data can be uniformly distributed into the buckets, the time complexity as below:
- step 2: O(n)
- step 3: O(m * (n/m) log (n/m)) ≈ O(n) when m is closed to n
- step 4: O(m)

So the total time complexity is O(n)
When the data is too big to fit into the memory, we can use bucket sort to split the data into small file firstly.
> https://www.geeksforgeeks.org/bucket-sort-2/

# Counting Sort #
It is just a special case of bucket sort, which the number of bucket equal to the range of input data, so we don't need to sort in each bucket.
1. Get/Change the data rang, suppose from 0 to k
2. Take a count array A to store the count of each unique object -> O(n)
3. Modify the count array A such that each element at each index stores the sum of previous counts -> O(k)
4. Output each object i from the input sequence based on the value of A[i] followed by decreasing A[i] by 1 -> O(n)

It can be used when k is not big, and the data range can be converted to non-negative integer.

> https://www.geeksforgeeks.org/counting-sort/
# Radix Sort #
The idea of Radix Sort is to do digit by digit sort starting from least significant digit to most significant digit. Radix sort uses counting sort/bucket sort as a subroutine to sort which needs to be stable.
> https://www.geeksforgeeks.org/radix-sort/

