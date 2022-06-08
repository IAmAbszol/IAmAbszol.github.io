# Study Journal for Data & Algorithm Problems.
---
## Kyle Darling

*The following sections and problems all reflect things I've learned while studying algorithms and began grouping certain problems together based on algorithm commonality. This doesn't denote that one algorithm fits all nor one algorithm is the only solution to set problem, these are completely biased views on my journey.*

# TODO: TABLE OF CONTENTS

## Dynamic Programming

One of my most sought after accomplishments has been to be able to tackle any dynamic programming problem that came my way. Hopefully below my explanations to the problems help, I always check my results and compare them to the discussions posted, especially using dynamic programming because the possibilities can almost seem endless. Each question will reflect a solution I enjoyed reading, helped use to solve the problem when stuck, or similar to my solutions.

### Algorithm Groups

Knapsack O/1 - Coin Change II
Keep/Flip - Maximum Alternating Subsequence Sum

### [Coin Change II](https://leetcode.com/problems/coin-change-2/)

Tasked with finding the number of unique ways we can form the given the amount, given the set of coins.

```
Input = [1,2,5]
Amount = 5
```

Instantly one could think of a brute-force like solution as to where we would recursively move down a tree, selecting either the same coin or a new coin each time.

**For example:**
```
Input = [1,2]
Amount = 2
```

At first glance the number of possibilities is 2. We're able to choose from either (1,1) or (2) ways. Next we could begin the recursive approach by moving down a tree starting at the highest number.

**Top-Down Approach**

![Recursion-Image](./docs/dynamicprogramming/coin_change_recursive.png)

> The code for this is quite simple.


```python
from functools import lru_cache

coins, amount = [1,2], 3
# i = index, a = amount

@lru_cache
def helper(i, a):
    if i < 0:
        return 0
    if a < 0:
        return 0
    if a == 0:
        return 1
    return helper(i - 1, a) + helper(i, a - coins[i])
print(f'Number of Unique Solutions {helper(len(coins) - 1, amount)}.')
```

    Number of Unique Solutions 2.


**Optimizing** 

The above solution using recursion + memoization is quite eloquent though we can reduce the time complexity down by introducing space into the mix, on other words an iterative solution.

**Bottom-Up Approach**

![Bottom-Up-Image](./docs/dynamicprogramming/coin_change_bottom_up.png)

> I decided to go ahead and optimize my code on space as well!

The current space requirement is O(a*n) where a is amount and n is number of coins. We can reduce this to O(a) by noticing that we only ever use one entire line, only referencing the top line to retrieve our previous result. Thus we can continually overwrite the **previous** result with our current which uses **previous**.


```python
from typing import List

def change(amount: int, coins: List[int]) -> int:
    dp = [0 for _ in range(amount + 1)]
    dp[0] = 1
    for i in coins:
        for j in range(1, amount + 1):
            if i <= j:
                dp[j] += dp[j - i]
    return dp[-1]
print(f'Number of Unique Solutions {change(amount, coins)}.')
```

    Number of Unique Solutions 2.


### [Maximum Alternating Subsequence Sum](https://leetcode.com/problems/maximum-alternating-subsequence-sum/)

This problem and one I had done afterwards kept getting me at the "subsequence" vs "subarray". 

**Subarray:** Is a contiguous, increasing in index partition of the main array.
**Subsequence:** Is a non-contiguous, increasing in index partition of the main array.

The main difference between the two is that the sum doesn't have to be contiguous but rather increasing.
[This page should help clear any confusion](https://medium.com/javarevisited/subarray-vs-subsequence-do-you-know-the-difference-3de0a2c2df52)

Now onto the problem!

We have to come up with a solution where the subsequence is maximized.

**Mistakes Made**

When I first tackled this problem I actually devised a way using O(n^2) time and space using Dynamic Programming to compute this with a contiguous subarray, not a subsequence, whoops!

Even with these kinds of problems we have to take into account that **Kadane's Algorithm** and others exist that handle maximizing our array/list.

**Rationalizing The Problem**

After failing epicly I took a step back and took what I had came up with recently. In the first attempt I had broken down indices *i* to *j* where i and j represent the start and end of the subarray. Inbetween i and j a rolling sum was used where:

1. ```if i == j, return nums[i]```
2. ```if |i - j| > 1, rolling_sum += nums[j] * (-1 ** j)```

In each case the previous row and column was used as a comparison between some maximization. 

Notice two things:
1. We calculate the rolling_sum on the go.
2. Any index could be even or odd if selected at the right time, meaning [1,2,3] we could have subsequences [1,2] or [2,3] and because of this 2 is both even and odd.

Therefore we can actually do this in O(n) time with O(1) space by computing at a given index it being odd or even.

The following below depicts everything needed for the problem and a practice problem from the examples.

![Maximum_Alternating_Subsequence_Sum_1](./docs/dynamicprogramming/maximum_alternating_subsequence_sum_1.png)
![Maximum_Alternating_Subsequence_Sum_2](./docs/dynamicprogramming/maximum_alternating_subsequence_sum_2.png)


```python
from typing import List

def maxAlternatingSum(nums: List[int]) -> int:
    odd, even = 0, 0
    for num in nums:
        odd, even = max(odd, even - num), max(even, odd + num)
    return even
print(f'Maximum value of the subsequence is {maxAlternatingSum([6,2,1,2,4,5])}.')
```

    Maximum value of the subsequence is 10.



```python

```
