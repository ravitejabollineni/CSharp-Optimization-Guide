# 136. Single Number Problem

## Problem Statement

Given a non-empty array of integers `nums`, every element appears twice except for one. Find that single one.

You must implement a solution with a linear runtime complexity and use only constant extra space.

### Examples

**Example 1:**
```
Input: nums = [2,2,1]
Output: 1
```

**Example 2:**
```
Input: nums = [4,1,2,1,2]
Output: 4
```

**Example 3:**
```
Input: nums = [1]
Output: 1
```

### Constraints

- `1 <= nums.length <= 3 * 10^4`
- `-3 * 10^4 <= nums[i] <= 3 * 10^4`
- Each element in the array appears twice except for one element which appears only once

---

## Solution

```csharp
public class Solution {
    public int SingleNumber(int[] nums) {
        int xor = 0;
        for(int i = 0; i < nums.Length; i++) {
            xor ^= nums[i];
        }
        return xor;
    }
}
```

---

## Explanation

### The XOR Approach

This solution uses the **XOR (exclusive OR)** bitwise operator, which has special mathematical properties that make it perfect for this problem.

### Key Properties of XOR

1. **Self-Cancellation:** `a ^ a = 0`
   - Any number XORed with itself equals 0
   - Example: `5 ^ 5 = 0`

2. **Identity Property:** `a ^ 0 = a`
   - Any number XORed with 0 remains unchanged
   - Example: `5 ^ 0 = 5`

3. **Commutative & Associative:** Order doesn't matter
   - `a ^ b ^ c = c ^ a ^ b`
   - Example: `1 ^ 2 ^ 3 = 3 ^ 1 ^ 2`

### How It Works

When you XOR all numbers in the array:
- Numbers that appear **twice** will cancel each other out (become 0)
- The number that appears **once** will remain

Think of it like a light switch:
- Press once: Light ON
- Press twice: Light OFF (back to original state)
- The number pressed an odd number of times (once) stays ON!

---

## Step-by-Step Example

Let's trace through `nums = [4, 1, 2, 1, 2]`

### Initial State
```
xor = 0
```

### Iteration 1: nums[0] = 4
```
xor = 0 ^ 4 = 4

Visual: [4]
```

### Iteration 2: nums[1] = 1
```
xor = 4 ^ 1 = 5

Visual: [4, 1]
```

### Iteration 3: nums[2] = 2
```
xor = 5 ^ 2 = 7

Visual: [4, 1, 2]
```

### Iteration 4: nums[3] = 1 ⚡
```
xor = 7 ^ 1 = 6

Visual: [4, 1, 2, 1]
The two 1's cancel out!
Result: [4, 2]
```

### Iteration 5: nums[4] = 2 ⚡
```
xor = 6 ^ 2 = 4

Visual: [4, 2, 2]
The two 2's cancel out!
Result: [4]

Final Answer: 4 ✓
```

### Complete XOR Chain
```
0 ^ 4 ^ 1 ^ 2 ^ 1 ^ 2

Rearranging (order doesn't matter):
0 ^ 4 ^ (1 ^ 1) ^ (2 ^ 2)
0 ^ 4 ^ 0 ^ 0
4 ✓
```

---

## Visual Bit-Level Example

Let's see how XOR works at the binary level with `nums = [2, 2, 1]`

```
Start:    xor = 0 = 0000

Step 1:   0000  (xor)
        ^ 0010  (2)
        ------
          0010

Step 2:   0010  (xor)
        ^ 0010  (2 again)
        ------
          0000  ← The two 2's cancelled out!

Step 3:   0000  (xor)
        ^ 0001  (1)
        ------
          0001 = 1 ✓
```

Notice how the duplicate 2's completely cancelled each other out (0010 ^ 0010 = 0000), leaving only the single 1.

---

## Complexity Analysis

### Time Complexity: O(n)
- We iterate through the array exactly once
- Each XOR operation is O(1)
- Total: O(n) where n is the length of the array

### Space Complexity: O(1)
- We only use one variable `xor` regardless of input size
- No additional data structures needed
- Constant extra space ✓

This meets both requirements of the problem!

---

## Why Other Approaches Don't Work

### ❌ Using a HashSet
```csharp
// Space: O(n) - violates constant space requirement
HashSet<int> set = new HashSet<int>();
foreach(int num in nums) {
    if(set.Contains(num)) set.Remove(num);
    else set.Add(num);
}
```
**Problem:** Uses O(n) extra space

### ❌ Sorting First
```csharp
// Time: O(n log n) - violates linear time requirement
Array.Sort(nums);
```
**Problem:** Sorting takes O(n log n) time

### ✅ XOR Approach
- Time: O(n) ✓
- Space: O(1) ✓
- Perfect solution!

---

## Real-World Analogy

### The Sock Matching Problem 🧦

Imagine you're doing laundry and matching socks:

```
You have: [Red, Blue, Green, Blue, Green]

Match pairs and remove them:
- Blue matches Blue → Remove both
- Green matches Green → Remove both
- Red has no match → This is your single sock!

Result: Red
```

XOR does exactly this - it "matches" and "removes" pairs, leaving only the unmatched item!

---

## Alternative Visualization: The Checklist ✓

```
Numbers to check off: 4, 1, 2, 1, 2

Start with empty list: [ ]

See 4: Add it     → [4]
See 1: Add it     → [4, 1]
See 2: Add it     → [4, 1, 2]
See 1: Already have 1! Cross both off → [4, 2]
See 2: Already have 2! Cross both off → [4]

What's left? 4!
```

---

## Test Cases

```csharp
// Test Case 1: Basic case
int[] test1 = {2, 2, 1};
Console.WriteLine(SingleNumber(test1)); // Output: 1

// Test Case 2: Middle element is unique
int[] test2 = {4, 1, 2, 1, 2};
Console.WriteLine(SingleNumber(test2)); // Output: 4

// Test Case 3: Single element
int[] test3 = {1};
Console.WriteLine(SingleNumber(test3)); // Output: 1

// Test Case 4: Negative numbers
int[] test4 = {-1, -1, -2};
Console.WriteLine(SingleNumber(test4)); // Output: -2

// Test Case 5: Large numbers
int[] test5 = {30000, 30000, 15000};
Console.WriteLine(SingleNumber(test5)); // Output: 15000
```

---

## Summary

The XOR solution is **elegant** and **optimal**:

✅ Uses a mathematical property (XOR self-cancellation)  
✅ Linear time complexity O(n)  
✅ Constant space complexity O(1)  
✅ Simple and clean code  
✅ Works with negative numbers  
✅ No sorting or extra data structures needed

**Key Insight:** XOR is like a perfect pairing mechanism - duplicates cancel out, singles remain!
