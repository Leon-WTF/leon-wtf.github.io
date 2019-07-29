---
title: "Simple Sort Algorithm"
category: Algorithm
tag: algorithm
---
# Bubble Sort #
Bubble Sort is the simplest sorting algorithm that works by repeatedly swapping the adjacent elements if they are in wrong order.  It can be optimized by stopping the algorithm if inner loop didn’t cause any swap.

> https://www.geeksforgeeks.org/bubble-sort/

# Insertion Sort #
Insertion sort is a simple sorting algorithm that works the way we sort playing cards in our hands. It can also be useful when input array is almost sorted, only few elements are misplaced in complete big array.
We can use binary search to reduce the number of comparisons in normal insertion sort. Binary Insertion Sort uses binary search to find the proper location to insert the selected item at each iteration. In normal insertion, sorting takes O(i) (at i-th iteration) in worst case. We can reduce it to O(log i) by using binary search. The algorithm, as a whole, still has a running worst case running time of $O(n^2)$ because of the series of swaps required for each insertion. 
> https://www.geeksforgeeks.org/insertion-sort/

# Selection Sort #
The selection sort algorithm first consider the data array as the combination of sorted part and unsorted part,  then sorts an array by repeatedly finding the minimum element (considering ascending order) from unsorted part and swap it to the beginning of unsorted part, it also begins the ending of sorted part.
The good thing about selection sort is it never makes more than O(n) swaps and can be useful when memory write is a costly operation.
It can be made stable if instead of swapping the minimum element to its position but by pushing every element one step forward.
> https://www.geeksforgeeks.org/selection-sort/

# Comparation #
|Algorithm name|Stable|In-place|Average Case Time complexity|Space complexity|
|--|--|--|--|--|--|
|Bubble Sort|Yes|Yes|$O(n^2)$|O(1)|
|Insertion Sort|Yes|Yes|$O(n^2)$|O(1)|
|Selection Sort|No|Yes|$O(n^2)$|O(1)|
|Quick Sort|No|Yes|O(nlogn)|O(1)|
|Merge Sort|Yes|No|O(nlogn)|O(n)|
|Bucket Sort|Yes|No|O(n)|O(n)|
|Counting Sort|Yes|No|O(n)|O(n+k)|
|Radix Sort|Yes|No|O(n)|O(n+k)/O(n)|
* k: the max value of the input data/the length of the counting array

> [Linear Sort Algorithm](https://leon-wtf.github.io/algorithm/2019/07/28/linear-sort/#more)
> [Quick Sort & Merge Sort](https://leon-wtf.github.io/algorithm/2019/07/26/quick-sort-merge-sort/#more)
