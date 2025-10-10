# Longest Subarray with Sum K

## 📖 Problem Statement

Given an array and a sum `k`, find the **length** of the longest subarray that sums to `k`.
### Examples

**Example 1:**
```
Input: array = [2, 3, 5], k = 5
Output: 2
Explanation: The subarray [2, 3] has sum = 5, length = 2
```

**Example 2:**
```
Input: array = [2, 3, 5, 1, 9], k = 10
Output: 3
Explanation: The subarray [2, 3, 5] has sum = 10, length = 3
```

---

## 🎯 What is a Subarray?

A **subarray** is a continuous part of an array.

### Visual Example:

```
Array: [2, 3, 5, 1, 9]

Valid subarrays (continuous):
✅ [2]
✅ [2, 3]
✅ [2, 3, 5]
✅ [3, 5, 1]
✅ [5, 1, 9]
✅ [2, 3, 5, 1, 9]  (entire array)

NOT subarrays (not continuous):
❌ [2, 5, 9]  (skipped 3)
❌ [3, 1]     (skipped 5)
```

---

## 🚀 Solution 1: Brute Force (Try Everything!)

### 💡 The Simple Idea

**"Let me check EVERY possible subarray and see which one sums to k!"**

Think of it like trying on every combination of clothes to find the perfect outfit! 👕👖

### 📝 Step-by-Step Approach

```
Step 1: Pick a starting point (i)
Step 2: Pick an ending point (j)
Step 3: Add all numbers from i to j
Step 4: If sum = k, check if this is the longest we've found
Step 5: Repeat for ALL possible starting and ending points
```

### 🎨 Visual Walkthrough

Let's use: `array = [2, 3, 5, 1, 9]`, `k = 10`

#### Starting at Index 0 (number 2):

```
Try [2]:                 sum = 2          ≠ 10 ❌
Try [2, 3]:              sum = 2+3 = 5    ≠ 10 ❌
Try [2, 3, 5]:           sum = 2+3+5 = 10  = 10 ✅ Length = 3
Try [2, 3, 5, 1]:        sum = 11         ≠ 10 ❌
Try [2, 3, 5, 1, 9]:     sum = 20         ≠ 10 ❌
```

#### Starting at Index 1 (number 3):

```
Try [3]:                 sum = 3          ≠ 10 ❌
Try [3, 5]:              sum = 3+5 = 8    ≠ 10 ❌
Try [3, 5, 1]:           sum = 3+5+1 = 9  ≠ 10 ❌
Try [3, 5, 1, 9]:        sum = 18         ≠ 10 ❌
```

#### Starting at Index 2 (number 5):

```
Try [5]:                 sum = 5          ≠ 10 ❌
Try [5, 1]:              sum = 5+1 = 6    ≠ 10 ❌
Try [5, 1, 9]:           sum = 15         ≠ 10 ❌
```

#### Starting at Index 3 (number 1):

```
Try [1]:                 sum = 1          ≠ 10 ❌
Try [1, 9]:              sum = 1+9 = 10    = 10 ✅ Length = 2
```

#### Starting at Index 4 (number 9):

```
Try [9]:                 sum = 9          ≠ 10 ❌
```

**Best Found:** Length = 3 from subarray [2, 3, 5]

### 📊 The Code (C# - Three Nested Loops)

```csharp
public class Solution {
    public int GetLongestSubarray(int[] nums, int k) {
        int n = nums.Length;
        int maxLen = 0;
        
        // Loop 1: Pick starting point
        for (int i = 0; i < n; i++) {
            
            // Loop 2: Pick ending point
            for (int j = i; j < n; j++) {
                
                // Loop 3: Calculate sum from i to j
                int sum = 0;
                for (int index = i; index <= j; index++) {
                    sum += nums[index];
                }
                
                // If sum equals k, update length
                if (sum == k) {
                    maxLen = Math.Max(maxLen, j - i + 1);
                }
            }
        }
        
        return maxLen;
    }
}
```

### 📝 Code Walkthrough

```csharp
// Example: nums = [2, 3, 5, 1, 9], k = 10

i=0, j=0: sum = 2           → 2 ≠ 10
i=0, j=1: sum = 2+3 = 5     → 5 ≠ 10
i=0, j=2: sum = 2+3+5 = 10  → 10 == 10 ✅ maxLen = 3

i=1, j=1: sum = 3           → 3 ≠ 10
i=1, j=2: sum = 3+5 = 8     → 8 ≠ 10
i=1, j=3: sum = 3+5+1 = 9   → 9 ≠ 10

... and so on
```

### 🐌 Why This is SLOW

**Three nested loops = Very slow!**

```
Array size = 5 elements

Loop 1 runs: 5 times
Loop 2 runs: 5 times for each i
Loop 3 runs: 5 times for each (i, j)

Total operations: 5 × 5 × 5 = 125 operations
```

**For 1000 elements:**
```
1000 × 1000 × 1000 = 1,000,000,000 operations! 😱
Your computer will cry! 💻😢
```

### ⏱️ Complexity

- **Time Complexity:** O(n³) - Cubic! Very slow!
- **Space Complexity:** O(1) - Good! Only uses a few variables

### ⚠️ Why Not Optimal?

1. ❌ Recalculates the same sums over and over
2. ❌ Too many nested loops
3. ❌ Wastes time on impossible combinations
4. ❌ Gets extremely slow with larger arrays

**Example of waste:**
```
We calculate [2, 3] sum
Then we calculate [2, 3, 5] from scratch
But we already knew [2, 3] = 5!
We could just add 5 to it! (5 + 5 = 10)
```

---

## 🔧 Solution 2: Slightly Better Brute Force

### 💡 Improvement

**"Instead of recalculating the sum each time, let's add as we go!"**

### 📊 The Code (C# - Two Nested Loops)

```csharp
public class Solution {
    public int GetLongestSubarray(int[] nums, int k) {
        int n = nums.Length;
        int maxLen = 0;
        
        // Loop 1: Pick starting point
        for (int i = 0; i < n; i++) {
            int sum = 0;  // Start fresh sum for each i
            
            // Loop 2: Extend the subarray
            for (int j = i; j < n; j++) {
                sum += nums[j];  // Just add the new element!
                
                if (sum == k) {
                    maxLen = Math.Max(maxLen, j - i + 1);
                }
            }
        }
        
        return maxLen;
    }
}
```

### 🎨 Visual Improvement

**Old way (3 loops):**
```csharp
i=0, j=2: 
  sum = 0;
  for (index = 0 to 2) {
    sum += nums[index];  // 2 + 3 + 5
  }
  // sum = 10 (had to add 3 numbers)
```

**New way (2 loops):**
```csharp
i=0, j=0: sum = 0 + 2 = 2
i=0, j=1: sum = 2 + 3 = 5      (just add 3!)
i=0, j=2: sum = 5 + 5 = 10     (just add 5!)
```

### 📝 Code Walkthrough

```csharp
// Example: nums = [2, 3, 5, 1, 9], k = 10

i=0: sum=0
  j=0: sum = 0+2 = 2     → 2 ≠ 10
  j=1: sum = 2+3 = 5     → 5 ≠ 10
  j=2: sum = 5+5 = 10    → 10 == 10 ✅ maxLen = 3
  j=3: sum = 10+1 = 11   → 11 ≠ 10
  j=4: sum = 11+9 = 20   → 20 ≠ 10

i=1: sum=0
  j=1: sum = 0+3 = 3     → 3 ≠ 10
  j=2: sum = 3+5 = 8     → 8 ≠ 10
  j=3: sum = 8+1 = 9     → 9 ≠ 10
  j=4: sum = 9+9 = 18    → 18 ≠ 10

... and so on
```

### ⏱️ Complexity

- **Time Complexity:** O(n²) - Still slow, but better!
- **Space Complexity:** O(1) - Good!

### ⚠️ Why Still Not Optimal?

1. ❌ Still checking too many unnecessary subarrays
2. ❌ No memory of what we've seen before
3. ❌ For 1000 elements: 1000 × 1000 = 1,000,000 operations (better, but still slow)

---

## 🚀 Solution 3: The SMART Way (Dictionary + Prefix Sum)

### 💡 The Brilliant Idea

**"What if we remember the sums we've seen before?"**

Think of it like breadcrumbs 🍞 - we leave markers as we walk, so we can find our way back!

### 🎓 New Concept: Prefix Sum

**Prefix Sum = Sum from the start up to current position**

```
Array:      [2,  3,  5,  1,  9]
Index:       0   1   2   3   4

Prefix Sum:
Index 0:  2                    = 2
Index 1:  2 + 3                = 5
Index 2:  2 + 3 + 5            = 10
Index 3:  2 + 3 + 5 + 1        = 11
Index 4:  2 + 3 + 5 + 1 + 9    = 20
```

### 🔑 The Magic Formula

```
If we want subarray sum = k:

Current sum - Previous sum = k
Therefore: Previous sum = Current sum - k

Example:
Current sum = 10, k = 10
Previous sum = 10 - 10 = 0
So we need a point where sum was 0 (the start!)
```

### 🎨 Visual Step-by-Step

**Array:** `[2, 3, 5, 1, 9]`, `k = 10`

#### Step 0: Setup

```csharp
Dictionary<int, int> prefixSumMap = new Dictionary<int, int>();
int sum = 0;
int maxLength = 0;
```

```
Dictionary (Memory): {}
Current sum: 0
Max length: 0
```

---

#### Step 1: Process index 0 (value = 2)

```csharp
sum = sum + nums[0];  // 0 + 2 = 2

// Check: Is sum == k?
if (sum == k)  // 2 == 10? NO ❌
    maxLength = 0 + 1;

// Check: Is (sum - k) in dictionary?
if (prefixSumMap.ContainsKey(sum - k))  // (2 - 10 = -8) in map? NO ❌
    // Not found

// Store in dictionary
if (!prefixSumMap.ContainsKey(sum))  // 2 not in map? YES
    prefixSumMap[sum] = 0;  // Store: {2: 0}
```

**Visual Memory:**
```
┌──────────┬────────┐
│   Sum    │ Index  │
├──────────┼────────┤
│    2     │   0    │
└──────────┴────────┘
```

---

#### Step 2: Process index 1 (value = 3)

```csharp
sum = sum + nums[1];  // 2 + 3 = 5

// Check: Is sum == k?
if (sum == k)  // 5 == 10? NO ❌
    maxLength = 1 + 1;

// Check: Is (sum - k) in dictionary?
if (prefixSumMap.ContainsKey(sum - k))  // (5 - 10 = -5) in map? NO ❌
    // Not found

// Store in dictionary
if (!prefixSumMap.ContainsKey(sum))  // 5 not in map? YES
    prefixSumMap[sum] = 1;  // Store: {2: 0, 5: 1}
```

**Visual Memory:**
```
┌──────────┬────────┐
│   Sum    │ Index  │
├──────────┼────────┤
│    2     │   0    │
│    5     │   1    │
└──────────┴────────┘
```

---

#### Step 3: Process index 2 (value = 5)

```csharp
sum = sum + nums[2];  // 5 + 5 = 10

// Check: Is sum == k?
if (sum == k)  // 10 == 10? YES! ✅
    maxLength = 2 + 1;  // maxLength = 3

// Check: Is (sum - k) in dictionary?
if (prefixSumMap.ContainsKey(sum - k))  // (10 - 10 = 0) in map? NO ❌
    // Not found

// Store in dictionary
if (!prefixSumMap.ContainsKey(sum))  // 10 not in map? YES
    prefixSumMap[sum] = 2;  // Store: {2: 0, 5: 1, 10: 2}
```

**Found:** Subarray from start to index 2: `[2, 3, 5]` = 10 ✅

**Visual:**
```
[2, 3, 5, 1, 9]
 └──────┘
Length = 3
```

---

#### Step 4: Process index 3 (value = 1)

```csharp
sum = sum + nums[3];  // 10 + 1 = 11

// Check: Is sum == k?
if (sum == k)  // 11 == 10? NO ❌
    maxLength = 3 + 1;

// Check: Is (sum - k) in dictionary?
if (prefixSumMap.ContainsKey(sum - k))  // (11 - 10 = 1) in map? NO ❌
    // Not found

// Store in dictionary
if (!prefixSumMap.ContainsKey(sum))  // 11 not in map? YES
    prefixSumMap[sum] = 3;  // Store: {2: 0, 5: 1, 10: 2, 11: 3}
```

---

#### Step 5: Process index 4 (value = 9)

```csharp
sum = sum + nums[4];  // 11 + 9 = 20

// Check: Is sum == k?
if (sum == k)  // 20 == 10? NO ❌
    maxLength = 4 + 1;

// Check: Is (sum - k) in dictionary?
if (prefixSumMap.ContainsKey(sum - k))  // (20 - 10 = 10) in map? YES! ✅
{
    int previousIndex = prefixSumMap[sum - k];  // previousIndex = 2
    int length = 4 - previousIndex;  // length = 4 - 2 = 2
    maxLength = Math.Max(maxLength, length);  // max(3, 2) = 3
}

// Store in dictionary
if (!prefixSumMap.ContainsKey(sum))  // 20 not in map? YES
    prefixSumMap[sum] = 4;
```

**What this means:**
```
sum at index 2 = 10
sum at index 4 = 20

Subarray from index 3 to 4: [1, 9]
Sum = 20 - 10 = 10 ✅

But length is only 2, so we keep the previous answer (3)
```

**Visual:**
```
[2, 3, 5, 1, 9]
          └───┘
       Length = 2 (shorter than 3, so we keep 3)
```

---

### 🎯 Why This Works - Real World Analogy

**The Piggy Bank Story 🐷💰**

```
You're saving money and recording your total savings:

Day 0: Have $2      Total: $2
Day 1: Add $3       Total: $5
Day 2: Add $5       Total: $10
Day 3: Add $1       Total: $11
Day 4: Add $9       Total: $20

Question: "When did I save exactly $10?"

Answer 1: From Day 0 to Day 2 (3 days) ✅
Answer 2: From Day 3 to Day 4 (2 days)
         Because $20 - $10 = $10

We want the longest period, so: 3 days!
```

---

### 📊 The Optimal Code (C#)

```csharp
public class Solution {
    public int LongestSubarray(int[] nums, int k) {
        // Our memory (Dictionary)
        Dictionary<int, int> prefixSumMap = new Dictionary<int, int>();
        
        int sum = 0;        // Running total
        int maxLength = 0;  // Best length found
        
        for (int i = 0; i < nums.Length; i++) {
            sum += nums[i];  // Add current number
            
            // Case 1: Sum from start equals k
            if (sum == k) {
                maxLength = i + 1;
            }
            
            // Case 2: Check if we've seen (sum - k) before
            if (prefixSumMap.ContainsKey(sum - k)) {
                int previousIndex = prefixSumMap[sum - k];
                int length = i - previousIndex;
                maxLength = Math.Max(maxLength, length);
            }
            
            // Save this sum (only first time we see it)
            if (!prefixSumMap.ContainsKey(sum)) {
                prefixSumMap[sum] = i;
            }
        }
        
        return maxLength;
    }
}
```

### 📝 Complete Code Walkthrough

```csharp
// Example: nums = [2, 3, 5, 1, 9], k = 10

// Initial state
Dictionary: {}
sum = 0
maxLength = 0

// i = 0, nums[0] = 2
sum = 2
sum != k (2 != 10)
(sum - k) = -8 not in Dictionary
Store: {2: 0}
maxLength = 0

// i = 1, nums[1] = 3
sum = 5
sum != k (5 != 10)
(sum - k) = -5 not in Dictionary
Store: {2: 0, 5: 1}
maxLength = 0

// i = 2, nums[2] = 5
sum = 10
sum == k (10 == 10) ✅
maxLength = 3
(sum - k) = 0 not in Dictionary
Store: {2: 0, 5: 1, 10: 2}
maxLength = 3

// i = 3, nums[3] = 1
sum = 11
sum != k (11 != 10)
(sum - k) = 1 not in Dictionary
Store: {2: 0, 5: 1, 10: 2, 11: 3}
maxLength = 3

// i = 4, nums[4] = 9
sum = 20
sum != k (20 != 10)
(sum - k) = 10 IS in Dictionary! ✅
previousIndex = 2
length = 4 - 2 = 2
maxLength = max(3, 2) = 3
Store: {2: 0, 5: 1, 10: 2, 11: 3, 20: 4}
maxLength = 3

// Return 3
```

---

### ⏱️ Complexity Analysis

**Time Complexity:** O(n) - ONE loop! Super fast! ⚡
```
Array size = 1000 elements
Operations = 1000 (just go through once!)

Compare to brute force:
1000 × 1000 × 1000 = 1,000,000,000
vs
1000

That's 1,000,000 times faster! 🚀
```

**Space Complexity:** O(n) - We use a Dictionary
```
In worst case, store n different sums
But it's worth it for the speed!
```

---

## 📊 Solution Comparison Table

| Solution | Loops | Time | Space | Speed for 1000 elements | Recommended? |
|----------|-------|------|-------|------------------------|--------------|
| **Brute Force (3 loops)** | 3 nested | O(n³) | O(1) | 1,000,000,000 operations 🐌 | ❌ Too slow |
| **Better Brute Force (2 loops)** | 2 nested | O(n²) | O(1) | 1,000,000 operations 🐢 | ❌ Still slow |
| **Dictionary + Prefix Sum** | 1 loop | O(n) | O(n) | 1,000 operations ⚡ | ✅ YES! |

---

## 🎓 Key Takeaways for Beginners

### What We Learned:

1. **Brute Force is OK to start** 👍
   - Easy to understand
   - Gets the job done for small inputs
   - Good for learning the problem

2. **But we can do better!** 🚀
   - Use smart data structures (Dictionary)
   - Remember what we've seen (Prefix sums)
   - Trade a little memory for LOTS of speed

3. **The "Trade-off" Concept** ⚖️
   ```
   Brute Force:  No extra memory, but SUPER slow
   Optimal:      Uses some memory, but SUPER fast
   
   In real life: Fast is almost always better!
   ```

4. **Dictionary is Your Friend** 📚
   - Store information as you go
   - Look it up instantly later
   - This pattern appears in MANY problems!

---

## 🧪 Test Cases

```csharp
public class TestCases {
    public static void Main() {
        Solution solution = new Solution();
        
        // Test Case 1: Basic case
        int[] test1 = {2, 3, 5};
        Console.WriteLine(solution.LongestSubarray(test1, 5));  
        // Output: 2 (subarray [2, 3])
        
        // Test Case 2: Longer array
        int[] test2 = {2, 3, 5, 1, 9};
        Console.WriteLine(solution.LongestSubarray(test2, 10)); 
        // Output: 3 (subarray [2, 3, 5])
        
        // Test Case 3: No subarray exists
        int[] test3 = {1, 2, 3};
        Console.WriteLine(solution.LongestSubarray(test3, 10)); 
        // Output: 0
        
        // Test Case 4: Entire array
        int[] test4 = {1, 2, 3, 4};
        Console.WriteLine(solution.LongestSubarray(test4, 10)); 
        // Output: 4 (entire array)
        
        // Test Case 5: Single element
        int[] test5 = {5};
        Console.WriteLine(solution.LongestSubarray(test5, 5));  
        // Output: 1
        
        // Test Case 6: Multiple valid subarrays
        int[] test6 = {1, 2, 3, 4, 5};
        Console.WriteLine(solution.LongestSubarray(test6, 9));  
        // Output: 3 (subarray [2, 3, 4])
    }
}
```

---

## 💡 Interview Tips

### When Solving This Problem:

1. **Start with Brute Force** 🎯
   ```
   "I can solve this with nested loops checking every subarray.
    That would be O(n²) or O(n³)."
   ```

2. **Mention the Optimization** 🚀
   ```
   "But we can do better using a Dictionary to store prefix sums.
    This brings it down to O(n) time with O(n) space."
   ```

3. **Explain the Trade-off** ⚖️
   ```
   "We're using extra memory to achieve much faster runtime,
    which is usually worth it in practice."
   ```

4. **Show You Understand Both** 📚
   ```
   Interviewers love when you know BOTH approaches
   and can explain WHY one is better!
   ```

---

## 🎉 Summary

### The Journey:

```
❌ Brute Force (3 loops):      O(n³) - Too slow!
⚠️  Better Brute Force (2 loops): O(n²) - Still slow!
✅ Dictionary + Prefix Sum:     O(n)  - Perfect! ⚡
```

### The Secret Sauce:

**Remember what you've seen!** Use a Dictionary to store prefix sums, then look them up instantly. This transforms a slow problem into a fast one!

---

## 🏆 Final Advice

**For Beginners:**
- Don't worry if the optimal solution seems hard at first
- Brute force helps you understand the problem
- Once you master this pattern (Dictionary + Prefix Sum), you'll see it EVERYWHERE!

**Practice Makes Perfect!** 💪

Try solving similar problems:
- Count subarrays with sum k
- Find subarray with sum 0
- Maximum length subarray with equal 0s and 1s

They all use the same technique! 🎯
