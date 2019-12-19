---
title: "Kth Smallest/Largest Element in ..."
catalog: true
toc_nav_num: true
date: 2019-12-16 17:30:51
subtitle: "Leet Code"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Swift

---

# Introduction

Here is the summary about the kth problems in LeetCode.

# Kth Largest Element in an Array

Find the kth largest element in an unsorted array. Note that it is the kth largest element in the sorted order, not the kth distinct element.

**Example 1:**

```
Input: [3,2,1,5,6,4] and k = 2
Output: 5
```

**Example 2:**

```
Input: [3,2,3,1,2,4,5,5,6] and k = 4
Output: 4
```

**Algorithm**

``` swift
class Solution {
    func findKthLargest(_ nums: [Int], _ k: Int) -> Int {
        func quickSort(_ nums: inout [Int], _ l: Int, _ r: Int) -> Int {
            var l = l, r = r
            let key = nums[l]
            while l < r {
                while l < r, key >= nums[r] {
                    r -= 1
                }
                nums[l] = nums[r]
                while l < r, key <= nums[l] {
                    l += 1
                }
                nums[r] = nums[l]
            }
            nums[l] = key
            return l
        }
        var l = 0, r = nums.count-1, nums = nums
        while l < r {
            let index = quickSort(&nums, l, r)
            switch index {
            case k - 1:
                return nums[index]
            case 0..<k-1:
                l = index + 1
                break
            default:
                r = index - 1
                break
            }
        }
        return nums[k-1]
    }
}
```

From leetCode [215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/)

# Kth Smallest Element in a BST

Given a binary search tree, write a function kthSmallest to find the kth smallest element in it.

**Note:**

You may assume k is always valid, 1 ≤ k ≤ BST's total elements.

**Example 1:**

```
Input: root = [3,1,4,null,2], k = 1
   3
  / \
 1   4
  \
   2
Output: 1
```

**Exapmle 2:**

```
Input: root = [5,3,6,2,4,null,null,1], k = 3
       5
      / \
     3   6
    / \
   2   4
  /
 1
Output: 3
```

**Algorithm:**

``` swift
public class TreeNode {
    public var val: Int
    public var left: TreeNode?
    public var right: TreeNode?
    public init(_ val: Int) {
        self.val = val
        self.left = nil
        self.right = nil
    }
}
class Solution {
    func kthSmallest(_ root: TreeNode?, _ k: Int) -> Int {
        var stack = [TreeNode]()
        var cur = root
        var k = k
        while cur != nil || !stack.isEmpty {
            if let temp = cur {
                stack.append(temp)
                cur = temp.left
            } else {
                let last = stack.removeLast()
                k -= 1
                if k == 0 {
                    return last.val
                }
                cur = last.right
            }
        }
        return 0
    }
}
```

From leetCode [230. Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/)

# Find K Pairs with Smallest Sums

You are given two integer arrays nums1 and nums2 sorted in ascending order and an integer k.

Define a pair (u,v) which consists of one element from the first array and one element from the second array.

Find the k pairs (u1,v1),(u2,v2) ...(uk,vk) with the smallest sums.

**Example 1:**

```
Input: nums1 = [1,7,11], nums2 = [2,4,6], k = 3
Output: [[1,2],[1,4],[1,6]] 
Explanation: The first 3 pairs are returned from the sequence: 
             [1,2],[1,4],[1,6],[7,2],[7,4],[11,2],[7,6],[11,4],[11,6]
```

**Example 2:**

```
Input: nums1 = [1,1,2], nums2 = [1,2,3], k = 2
Output: [1,1],[1,1]
Explanation: The first 2 pairs are returned from the sequence: 
             [1,1],[1,1],[1,2],[2,1],[1,2],[2,2],[1,3],[1,3],[2,3]
```

**Example 3:**

```
Input: nums1 = [1,2], nums2 = [3], k = 3
Output: [1,3],[2,3]
Explanation: All possible pairs are returned from the sequence: [1,3],[2,3]
```

**Algorithm:**

``` swift
class Solution {
    func kSmallestPairs(_ nums1: [Int], _ nums2: [Int], _ k: Int) -> [[Int]] {
        guard !nums1.isEmpty, !nums2.isEmpty, k >= 0 else {
            return []
        }
        func insert(_ arr: inout [(Int, Int, Int)], _ new: (Int, Int, Int)) {
            var s = 0, e = arr.count-1
            while s <= e {
                let mid = (s + e) >> 1
                if (mid == 0 && new.0 <= arr[mid].0) ||
                    (mid > 0 && new.0 >= arr[mid-1].0 && new.0 <= arr[mid].0) {
                    arr.insert(new, at: mid)
                    return
                } else if new.0 > arr[mid].0 {
                    s = mid + 1
                } else {
                    e = mid - 1
                }
            }
            arr.insert(new, at: arr.count)
        }
        var arr = [(Int, Int, Int)]()
        for i in nums1.indices {
            arr.append((nums1[i]+nums2[0], i, 0))
        }
        var res = [[Int]]()
        while !arr.isEmpty, res.count < k {
            let (_, i, j) = arr.removeFirst()
            res.append([nums1[i], nums2[j]])
            if j < nums2.count-1 {
                insert(&arr, (nums1[i]+nums2[j+1], i, j+1))
            }
        }
        return res
    }
}
```

**Comment:**

Here is a simple example demonstrate how this algorithm works:

![example](/img/article/20191216/1.png)

From leetCode [373. Find K Pairs with Smallest Sums](https://leetcode.com/problems/find-k-pairs-with-smallest-sums/)

# Kth Smallest Element in a Sorted Matrix

Given a n x n matrix where each of the rows and columns are sorted in ascending order, find the kth smallest element in the matrix.

Note that it is the kth smallest element in the sorted order, not the kth distinct element.

**Example:**

```
matrix = [
   [ 1,  5,  9],
   [10, 11, 13],
   [12, 13, 15]
],
k = 8,

return 13.
```

**Algorithm:**

``` swift
class Solution {
    func kthSmallest(_ matrix: [[Int]], _ k: Int) -> Int {
        guard !matrix.isEmpty, !matrix.first!.isEmpty, k > 0 else {
            return 0
        }
        func insert(_ arr: inout [(Int, Int, Int)], _ new: (Int, Int, Int)) {
            var s = 0, e = arr.count-1
            while s <= e {
                let mid = (s + e) >> 1
                if (mid == 0 && new.0 <= arr[mid].0) ||
                    (mid > 0 && new.0 > arr[mid-1].0 && new.0 <= arr[mid].0) {
                    arr.insert(new, at: mid)
                    return
                } else if new.0 > arr[mid].0 {
                    s = mid + 1
                } else {
                    e = mid - 1
                }
            }
            arr.insert(new, at: arr.count)
        }
        var arr = [(Int, Int, Int)]()
        for i in matrix.indices {
            arr.append((matrix[i][0], i, 0))
        }
        var k = k, i = 0, j = 0
        while !arr.isEmpty, k > 0 {
            (_, i, j) = arr.removeFirst()
            if j < matrix.first!.count-1 {
                insert(&arr, (matrix[i][j+1], i, j+1))
            }
            k -= 1
        }
        return matrix[i][j]
    }
}
```

``` swift
class Solution {
    func kthSmallest(_ matrix: [[Int]], _ k: Int) -> Int {
        guard !matrix.isEmpty else {
            return 0
        }
        let n = matrix.count-1
        var lo = matrix[0][0], hi = matrix[n][n]
        while lo < hi {
            let mid = (lo + hi) >> 1
            var count = 0, j = n
            for i in 0...n {
                while j >= 0, matrix[i][j] > mid {
                    j -= 1
                }
                count += j + 1
            }
            if count < k {
                lo = mid + 1
            } else {
                hi = mid
            }
        }
        return lo
    }
}
```

From leetCode [378. Kth Smallest Element in a Sorted Matrix](https://leetcode.com/problems/kth-smallest-element-in-a-sorted-matrix/)

# Kth Smallest Number in Multiplication Table

Nearly every one have used the Multiplication Table. But could you find out the k-th smallest number quickly from the multiplication table?

Given the height m and the length n of a m * n Multiplication Table, and a positive integer k, you need to return the k-th smallest number in this table.

**Example 1:**

```
Input: m = 3, n = 3, k = 5
Output: 
Explanation: 
The Multiplication Table:
1	2	3
2	4	6
3	6	9

The 5-th smallest number is 3 (1, 2, 2, 3, 3).
```

**Example 2:**

```
Input: m = 2, n = 3, k = 6
Output: 
Explanation: 
The Multiplication Table:
1	2	3
2	4	6

The 6-th smallest number is 6 (1, 2, 2, 3, 4, 6).
```

**Note:**

1. The m and n will be in the range [1, 30000].
2. The k will be in the range [1, m * n]

**Algorithm:**

``` swift
class Solution {
    func findKthNumber(_ m: Int, _ n: Int, _ k: Int) -> Int {
        guard m > 0, n > 0, k > 0 else {
            return 0
        }
        var lo = 1, hi = m * n
        while lo < hi {
            let mid = (lo + hi) >> 1
            var count = 0, nN = n
            for i in 1...m {
                while nN > 0, i * nN > mid {
                    nN -= 1
                }
                count += nN
            }
            if count < k {
                lo = mid + 1
            } else {
                hi = mid
            }
        }
        return lo
    }
}
```

From leetCode [668. Kth Smallest Number in Multiplication Table](https://leetcode.com/problems/kth-smallest-number-in-multiplication-table/)

# Find K-th Smallest Pair Distance

Given an integer array, return the k-th smallest distance among all the pairs. The distance of a pair (A, B) is defined as the absolute difference between A and B.

**Example 1:**

```
Input:
nums = [1,3,1]
k = 1
Output: 0 
Explanation:
Here are all the pairs:
(1,3) -> 2
(1,1) -> 0
(3,1) -> 2
Then the 1st smallest distance pair is (1,1), and its distance is 0.
```

**Note:**

1. 2 <= len(nums) <= 10000.
2. 0 <= nums[i] < 1000000.
3. 1 <= k <= len(nums) * (len(nums) - 1) / 2.

**Algorithm:**

``` swift
class Solution {
    func smallestDistancePair(_ nums: [Int], _ k: Int) -> Int {
        guard nums.count > 1 else {
            return 0
        }
        let sortedNums = nums.sorted()
        var lo = 0, hi = sortedNums.last! - sortedNums.first!
        while lo < hi {
            let mid = (lo + hi) >> 1
            var count = 0, i = 0
            for j in 0..<sortedNums.count {
                while i < sortedNums.count, sortedNums[j] - sortedNums[i] > mid {
                    i += 1
                }
                count += j-i
            }
            if count < k {
                lo = mid + 1
            } else {
                hi = mid
            }
        }
        return lo
    }
}
```

From leetCode [719. Find K-th Smallest Pair Distance](https://leetcode.com/problems/find-k-th-smallest-pair-distance/)

# K-th Smallest Prime Fraction

A sorted list A contains 1, plus some number of primes.  Then, for every p < q in the list, we consider the fraction p/q.

What is the K-th smallest fraction considered?  Return your answer as an array of ints, where answer[0] = p and answer[1] = q.

**Example:**

```
Input: A = [1, 2, 3, 5], K = 3
Output: [2, 5]
Explanation:
The fractions to be considered in sorted order are:
1/5, 1/3, 2/5, 1/2, 3/5, 2/3.
The third fraction is 2/5.

Input: A = [1, 7], K = 1
Output: [1, 7]
```

**Note:**

- A will have length between 2 and 2000.
- Each A[i] will be between 1 and 30000.
- K will be between 1 and A.length * (A.length - 1) / 2.

**Algorithm:**

``` swift
class Solution {
    func kthSmallestPrimeFraction(_ A: [Int], _ K: Int) -> [Int] {
        let n = A.count
        guard n > 1, K > 0 else {
            return []
        }
        var lo = 0.0, hi = 1.0
        while lo < hi {
            let mid = (lo + hi) / 2
            var j = 1, count = 0, maxVal = 0.0, res = (0, 0)
            for i in 0..<n-1 {
                while j < n, Double(A[i]) > mid * Double(A[j]) {
                    j += 1
                }
                if n == j {
                    break
                }
                count += n-j
                let val = Double(A[i]) / Double(A[j])
                if val > maxVal {
                    res = (i, j)
                    maxVal = val
                }
            }
            if count < K {
                lo = mid
            } else if count == K {
                return [A[res.0], A[res.1]]
            } else {
                hi = mid
            }
        }
        return []
    }
}
```

From leetCode [786. K-th Smallest Prime Fraction](https://leetcode.com/problems/k-th-smallest-prime-fraction/)

# Minimum Cost to Hire K Workers

There are N workers.  The i-th worker has a quality[i] and a minimum wage expectation wage[i].

Now we want to hire exactly K workers to form a paid group.  When hiring a group of K workers, we must pay them according to the following rules:
1. Every worker in the paid group should be paid in the ratio of their quality compared to other workers in the paid group.
2. Every worker in the paid group must be paid at least their minimum wage expectation.

Return the least amount of money needed to form a paid group satisfying the above conditions.

**Example 1:**

```
Input: quality = [10,20,5], wage = [70,50,30], K = 2
Output: 105.00000
Explanation: We pay 70 to 0-th worker and 35 to 2-th worker.
```

**Example 2:**

```
Input: quality = [3,1,10,10,1], wage = [4,8,2,2,7], K = 3
Output: 30.66667
Explanation: We pay 4 to 0-th worker, 13.33333 to 2-th and 3-th workers seperately. 
```

**Note:**

1. 1 <= K <= N <= 10000, where N = quality.length = wage.length
2. 1 <= quality[i] <= 10000
3. 1 <= wage[i] <= 10000
4. Answers within 10^-5 of the correct answer will be considered correct.

**Algorithm:**

``` swift
class Solution {
    func mincostToHireWorkers(_ quality: [Int], _ wage: [Int], _ K: Int) -> Double {
        func insert<T: Comparable>(_ new: (T, Int), _ arr: inout [(T, Int)]) {
            var l = 0, r = arr.count-1
            while l <= r {
                let mid = (l+r) >> 1
                if (mid == 0 && new.0 <= arr[mid].0) ||
                    (mid > 0 && new.0 <= arr[mid].0 && new.0 > arr[mid-1].0) {
                    arr.insert(new, at: mid)
                    return
                } else if new.0 > arr[mid].0 {
                    l = mid + 1
                } else {
                    r = mid - 1
                }
            }
            arr.insert(new, at: arr.count)
        }
        var arr = [(Double, Int)]()
        for i in quality.indices {
            insert((Double(wage[i]) / Double(quality[i]), i), &arr)
        }
        var res = pow(10.0, 10.0)
        var qualities = [(Int, Int)]()
        var sump = 0
        for i in 0..<arr.count {
            let temp = quality[arr[i].1]
            insert((temp, arr[i].1), &qualities)
            sump += temp
            if qualities.count > K {
                sump -= qualities.popLast()!.0
            }
            if qualities.count == K {
                let ratio = arr[i].0
                res = min(res, Double(sump) * ratio)
            }
        }
        return res
    }
}
```

From leetCode [857. Minimum Cost to Hire K Workers](https://leetcode.com/problems/minimum-cost-to-hire-k-workers/)