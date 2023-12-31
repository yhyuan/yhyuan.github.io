---
title: 389. find the difference
date: '2022-02-13'
tags: ['leetcode', 'rust', 'easy']
draft: false
description: Solution for leetcode 0389 find the difference
---

 

  You are given two strings s and t.

  String t is generated by random shuffling string s and then add one more letter at a random position.

  Return the letter that was added to t.

   

 >   Example 1:

  

 >   Input: s <TeX>=</TeX> "abcd", t <TeX>=</TeX> "abcde"

 >   Output: "e"

 >   Explanation: 'e' is the letter that was added.

  

 >   Example 2:

  

 >   Input: s <TeX>=</TeX> "", t <TeX>=</TeX> "y"

 >   Output: "y"

  

 >   Example 3:

  

 >   Input: s <TeX>=</TeX> "a", t <TeX>=</TeX> "aa"

 >   Output: "a"

  

 >   Example 4:

  

 >   Input: s <TeX>=</TeX> "ae", t <TeX>=</TeX> "aea"

 >   Output: "a"

  

   

  **Constraints:**

  

 >   	0 <TeX>\leq</TeX> s.length <TeX>\leq</TeX> 1000

 >   	t.length <TeX>=</TeX><TeX>=</TeX> s.length + 1

 >   	s and t consist of lower-case English letters.


## Solution
### Rust
```rust
pub struct Solution {}


// submission codes start here

impl Solution {
    pub fn find_the_difference(s: String, t: String) -> char {
        let res = s.chars()
            .chain(t.chars())
            .map(|ch| ch as u8)
            .fold(0u8, |sum, val| sum ^ val);
        res as char
    }
}

// submission codes end

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_389() {
        assert_eq!(Solution::find_the_difference("abcd".to_string(), "abcde".to_string()), 'e');
    }
}

```
