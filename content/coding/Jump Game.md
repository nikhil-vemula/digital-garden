---
title: Jump Game
draft: false
tags:
---
```python
class Solution:
    def canJump(self, nums: List[int]) -> bool:
        goal = len(nums) - 1
        for i in range(len(nums)-1, -1, -1):
            if i + nums[i] >= goal:
                goal = i
        return goal == 0

        # @cache
        # def dfs(i):
        #     print(i)
        #     if i == len(nums)-1:
        #         return True
            
        #     for j in range(1, nums[i]+1):
        #         if dfs(i + j):
        #             return True
            
        #     return False
        
        # return dfs(0)
```