---
title: "Sliding Window algorithm - substring search problems"
catalog: true
toc_nav_num: true
date: 2019-10-07 18:39:23
subtitle: "Leet Code"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Swift

---

# Introduction

Sliding Window algorithm template to solve almost Leetcode substring search problem.

# Find All Anagrams in a String

Given a string **s** and a **non-empty** string **p**, find all the start indices of **p**'s anagrams in **s**.

Strings consists of lowercase English letters only and the length of both strings **s** and **p**will not be larger than 20,100.

The order of output does not matter.

**Example 1:**
```
Input:
s: "cbaebabacd" p: "abc"

Output:
[0, 6]

Explanation:
The substring with start index = 0 is "cba", which is an anagram of "abc".
The substring with start index = 6 is "bac", which is an anagram of "abc".
```

**Example 2:**
```
Input:
s: "abab" p: "ab"

Output:
[0, 1, 2]

Explanation:
The substring with start index = 0 is "ab", which is an anagram of "ab".
The substring with start index = 1 is "ba", which is an anagram of "ab".
The substring with start index = 2 is "ab", which is an anagram of "ab".
```

**Algorithm:**
``` swift
class Solution {
    func findAnagrams(_ s: String, _ p: String) -> [Int] {
        guard !s.isEmpty, !p.isEmpty, s.count >= p.count else {
            return []
        }
        var map = [Character: Int]()
        for c in p {
            map.updateValue(map[c, default: 0] + 1, forKey: c)
        }
        var start = 0, end = 0
        var result = [Int]()
        var p = Array(p), s = Array(s)
        var counter = map.count
        while end < s.count {
            let sE = s[end]
            if let mapValue = map[sE] {
                map.updateValue(mapValue-1, forKey: sE)
                if mapValue == 1 {
                    counter -= 1
                }
            }
            end += 1
            while counter == 0 {
                let sS = s[start]
                if let mapValue = map[sS] {
                    map.updateValue(mapValue+1, forKey: sS)
                    if mapValue >= 0 {
                        counter += 1
                    }
                }
                if end-start == p.count {
                    result.append(start)
                }
                start += 1
            }
        }
        return result
    }
}
```

From leetCode [438. Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/)

# Minimum Window Substring

Given a string S and a string T, find the minimum window in S which will contain all the characters in T in complexity **O(n)**.

**Example:**
```
Input: S = "ADOBECODEBANC", T = "ABC"
Output: "BANC"
```

**Note:**
- If there is no such window in S that convers all characters in T, return the empty string "".
- If there is such window, you are guaranteed that there will always be only one unique minimum window in S.

**algorithm:**
``` swift
class Solution {
    func minWindow(_ s: String, _ t: String) -> String {
        guard !s.isEmpty, !t.isEmpty, s.count >= t.count else {
            return ""
        }
        var map = [Character: Int]()
        for tS in t {
            map.updateValue(map[tS, default: 0] + 1, forKey: tS)
        }
        var start = 0, end = 0, counter = map.count
        let sA = Array(s)
        var result = (-1, s.count+1)
        while end < sA.count {
            let sE = sA[end]
            if let mapValue = map[sE] {
                map.updateValue(mapValue-1, forKey: sE)
                if mapValue == 1 {
                    counter -= 1
                }
            }
            end += 1
            while counter == 0 {
                let sS = sA[start]
                if let mapValue = map[sS] {
                    map.updateValue(mapValue+1, forKey: sS)
                    if mapValue >= 0 {
                        counter += 1
                    }
                }
                if end-start < result.1 {
                    result = (start, end-start)
                }
                start += 1
            }
        }
        return result.0 == -1 ? "" : String(s[s.index(s.startIndex, offsetBy: result.0)..<s.index(s.startIndex, offsetBy: result.0+result.1)])
    }
}
```

From leetCode [76. Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/)

# Longest Substring Without Repeating Characters

Given a string, find the length of the **longest substring** without repeating characters.

**Example 1:**
```
Input: "abcabcbb"
Output: 3 
Explanation: The answer is "abc", with the length of 3. 
```

**Example 2:**
```
Input: "bbbbb"
Output: 1
Explanation: The answer is "b", with the length of 1.
```

**Example 3:**
```
Input: "pwwkew"
Output: 3
Explanation: The answer is "wke", with the length of 3. 
Note: that the answer must be a substring, "pwke" is a subsequence and not a substring.
```

**Algorithm:**
``` swift
class Solution {
    func lengthOfLongestSubstring(_ s: String) -> Int {
        guard !s.isEmpty else {
            return 0
        }
        let sA = Array(s)
        var map = [Character: Int]()
        var start = 0, end = 0, counter = 0, result = 0
        while end < sA.count {
            let sE = sA[end]
            if let oldValue = map.updateValue(map[sE, default: 0] + 1, forKey: sE), oldValue > 0 {
                counter += 1
            }
            end += 1
            while counter > 0 {
                let sS = sA[start]
                if let oldValue = map.updateValue(map[sS, default: 0] - 1, forKey: sS), oldValue > 1 {
                    counter -= 1
                }
                start += 1
            }
            result = max(result, end-start)
        }
        return result
    }
}
```

From leetCode [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)

# Substring with Concatenation of All Words

You are given a string, **s**, and a list of words, **words**, that are all of the same length. Find all starting indices of substring(s) in **s** that is a concatenation of each words exactly once and without any intervening characters.

**Example 1:**
```
Input:
  s = "barfoothefoobarman",
  words = ["foo","bar"]
Output: [0,9]
Explanation: Substrings starting at index 0 and 9 are "barfoor" and "foobar" respectively.
The output order does not matter, returning [9,0] is fine too.
```

**Example 2:**
```
Input:
  s = "wordgoodgoodgoodbestword",
  words = ["word","good","best","word"]
Output: []
```

**Algorithm:**
``` swift
class Solution {
    func findSubstring(_ s: String, _ words: [String]) -> [Int] {
        guard !s.isEmpty, !words.isEmpty, s.count >= words.first!.count else {
            return []
        }
        var map = [String: Int]()
        for word in words {
            map.updateValue(map[word, default: 0] + 1, forKey: word)
        }
        let sA = Array(s)
        var curMap = [String: Int]()
        let wl = words.first!.count
        var result = [Int]()
        for i in 0..<wl {
            var start = i
            var count = 0
            for j in stride(from: i, through: s.count-wl, by: wl) {
                let str = String(sA[j..<j+wl])
                if let mapValue = map[str] {
                    curMap.updateValue(curMap[str, default: 0] + 1, forKey: str)
                    if curMap[str]! <= mapValue {
                        count += 1
                    }
                    
                    while curMap[str]! > map[str]! {
                        let tmp = String(sA[start..<start+wl])
                        curMap.updateValue(curMap[tmp, default: 0] - 1, forKey: tmp)
                        start += wl
                        if curMap[tmp]! < map[tmp]! {
                            count -= 1
                        }
                    }
                    if count == words.count {
                        result.append(start)
                        let tmp = String(sA[start..<start+wl])
                        curMap.updateValue(curMap[tmp, default: 0] - 1, forKey: tmp)
                        start += wl
                        count -= 1
                    }
                } else {
                    start = j + wl
                    count = 0
                    curMap.removeAll()
                }
            }
            curMap.removeAll()
        }
        return result
    }
}
```

From leetCode [30. Substring with Concatenation of All Words](https://leetcode.com/problems/substring-with-concatenation-of-all-words/)