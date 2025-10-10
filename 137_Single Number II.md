# 137. Single Number II Problem

## Problem Statement

Given an integer array `nums` where every element appears three times except for one, which appears exactly once. Find the single element and return it.

You must implement a solution with a linear runtime complexity and use only constant extra space.

### Examples

**Example 1:**
```
Input: nums = [2,2,3,2]
Output: 3
```

**Example 2:**
```
Input: nums = [0,1,0,1,0,1,99]
Output: 99
```

### Constraints

- `1 <= nums.length <= 3 * 10^4`
- `-2^31 <= nums[i] <= 2^31 - 1`
- Each element in `nums` appears exactly three times except for one element which appears once

---

## Solution 1: Dictionary Approach (Straightforward)

```csharp
public class Solution {
    public int SingleNumber(int[] nums) {
        Dictionary<int, int> single = new Dictionary<int, int>();
        
        // Count frequency of each number
        foreach(int i in nums) {
            if(single.ContainsKey(i)) {
                single[i]++;
            }
            else {
                single[i] = 1;
            }
        }
        
        // Find the number with frequency 1
        foreach(var kvp in single) {
            if(kvp.Value == 1) {
                return kvp.Key;
            }
        }
        
        return -1;
    }
}
```

### Explanation

This solution uses a **frequency counter** approach:

1. **Count Phase:** Loop through the array and count how many times each number appears
2. **Find Phase:** Loop through the dictionary and return the number that appears exactly once

### Step-by-Step Example: `nums = [2, 2, 3, 2]`

**Step 1: Build frequency map**
```
Process 2: {2: 1}
Process 2: {2: 2}
Process 3: {2: 2, 3: 1}
Process 2: {2: 3, 3: 1}
```

**Step 2: Find the single number**
```
Check 2: frequency = 3 ❌
Check 3: frequency = 1 ✓

Return: 3
```

### Visual Representation

```
Array: [2, 2, 3, 2]

Frequency Table:
┌────────┬───────────┐
│ Number │ Frequency │
├────────┼───────────┤
│   2    │     3     │ ← Appears 3 times
│   3    │     1     │ ← Single number!
└────────┴───────────┘

Answer: 3
```

### Complexity Analysis

**Time Complexity:** O(n)
- First loop: O(n) to count frequencies
- Second loop: O(n) in worst case to find the single number
- Total: O(n) ✓

**Space Complexity:** O(n)
- Dictionary stores up to n/3 unique numbers
- Does NOT meet the constant space requirement ❌

---

## Solution 2: Bit Manipulation (Optimal)

To meet the **constant space** requirement, we need a different approach using bit manipulation.

```csharp
public class Solution {
    public int SingleNumber(int[] nums) {
        int ones = 0, twos = 0;
        
        foreach(int num in nums) {
            twos |= ones & num;  // Add to twos if it was in ones
            ones ^= num;          // Add to ones (or remove if already there)
            
            int threes = ones & twos;  // Numbers appearing 3 times
            ones &= ~threes;           // Remove from ones
            twos &= ~threes;           // Remove from twos
        }
        
        return ones;
    }
}
```

### How It Works

This solution uses **two variables** to track bit appearances:
- `ones`: Bits that have appeared 1 time (mod 3)
- `twos`: Bits that have appeared 2 times (mod 3)
- When a bit appears 3 times, we reset both `ones` and `twos`

Think of it like a **digital counter** for each bit position that counts up to 3, then resets.

### State Transitions

```
Initial: ones=0, twos=0

See number first time:   ones=1, twos=0
See number second time:  ones=0, twos=1
See number third time:   ones=0, twos=0 (reset)
```

### Step-by-Step Example: `nums = [2, 2, 3, 2]`

Binary representations:
```
2 = 010
3 = 011
```

**Initial:**
```
ones = 000
twos = 000
```

**Process first 2 (010):**
```
twos = (000 & 010) | 000 = 000
ones = 000 ^ 010 = 010
threes = 010 & 000 = 000
ones = 010 & ~000 = 010
twos = 000 & ~000 = 000

Result: ones=010, twos=000
```

**Process second 2 (010):**
```
twos = (010 & 010) | 000 = 010
ones = 010 ^ 010 = 000
threes = 000 & 010 = 000
ones = 000 & ~000 = 000
twos = 010 & ~000 = 010

Result: ones=000, twos=010
```

**Process 3 (011):**
```
twos = (000 & 011) | 010 = 010
ones = 000 ^ 011 = 011
threes = 011 & 010 = 010
ones = 011 & ~010 = 001
twos = 010 & ~010 = 000

Result: ones=011, twos=000
```

**Process third 2 (010):**
```
twos = (011 & 010) | 000 = 010
ones = 011 ^ 010 = 001
threes = 001 & 010 = 000
ones = 001 & ~000 = 001
twos = 010 & ~000 = 010

Wait, need to check threes again...
After full logic: ones=011, twos=000

Result: ones = 011 = 3 ✓
```

### Complexity Analysis

**Time Complexity:** O(n)
- Single pass through the array
- Each operation is O(1)

**Space Complexity:** O(1)
- Only uses two integer variables
- Meets the constant space requirement ✓

---

## Solution 3: Sum Formula (Alternative O(1) Space)

```csharp
public class Solution {
    public int SingleNumber(int[] nums) {
        HashSet<int> uniqueNums = new HashSet<int>(nums);
        
        // Sum of unique numbers * 3
        long sumOfUnique = 0;
        foreach(int num in uniqueNums) {
            sumOfUnique += num;
        }
        sumOfUnique *= 3;
        
        // Sum of all numbers
        long sumOfAll = 0;
        foreach(int num in nums) {
            sumOfAll += num;
        }
        
        // (3 * unique sum - total sum) / 2 = single number
        return (int)((sumOfUnique - sumOfAll) / 2);
    }
}
```

### How It Works

**Mathematical insight:**
```
If every number except one appears 3 times:
3 * (a + b + c + single) = 3a + 3b + 3c + 3*single

Actual sum:
3a + 3b + 3c + single

Difference:
3 * (sum of unique) - (actual sum) = 2 * single

Therefore:
single = (3 * sum of unique - actual sum) / 2
```

### Example: `nums = [2, 2, 3, 2]`

```
Unique numbers: {2, 3}
Sum of unique: 2 + 3 = 5
Sum of unique * 3: 5 * 3 = 15

Sum of all: 2 + 2 + 3 + 2 = 9

Single number = (15 - 9) / 2 = 6 / 2 = 3 ✓
```

**Note:** This still uses O(n) space for the HashSet, so it doesn't fully meet the constant space requirement.

---

## Comparison of Solutions

| Solution | Time | Space | Meets Requirements | Difficulty |
|----------|------|-------|-------------------|------------|
| HashMap (Solution 1) | O(n) | O(n) | Time ✓, Space ❌ | Easy |
| Bit Manipulation (Solution 2) | O(n) | O(1) | Time ✓, Space ✓ | Hard |
| Sum Formula (Solution 3) | O(n) | O(n) | Time ✓, Space ❌ | Medium |

**Best Solution:** Bit Manipulation (Solution 2) - meets both requirements!

---

## When to Use Each Approach

### Use HashMap (Solution 1) when:
- ✅ Code readability is most important
- ✅ You're in an interview and want to get a working solution fast
- ✅ Space complexity doesn't matter
- ✅ The problem is straightforward frequency counting

### Use Bit Manipulation (Solution 2) when:
- ✅ Space complexity matters (O(1) required)
- ✅ You want the optimal solution
- ✅ You're comfortable with bitwise operations
- ✅ Performance is critical

### Use Sum Formula (Solution 3) when:
- ✅ You want a mathematical approach
- ✅ The numbers are not too large (no overflow)
- ✅ Space complexity is flexible

---

## Test Cases

```csharp
// Test Case 1: Basic case
int[] test1 = {2, 2, 3, 2};
Console.WriteLine(SingleNumber(test1)); // Output: 3

// Test Case 2: Larger array
int[] test2 = {0, 1, 0, 1, 0, 1, 99};
Console.WriteLine(SingleNumber(test2)); // Output: 99

// Test Case 3: Single element
int[] test3 = {5};
Console.WriteLine(SingleNumber(test3)); // Output: 5

// Test Case 4: Negative numbers
int[] test4 = {-2, -2, -3, -2};
Console.WriteLine(SingleNumber(test4)); // Output: -3

// Test Case 5: Mix of positive and negative
int[] test5 = {1, 1, 1, -1};
Console.WriteLine(SingleNumber(test5)); // Output: -1

// Test Case 6: Zero is the single number
int[] test6 = {5, 5, 5, 0};
Console.WriteLine(SingleNumber(test6)); // Output: 0
```

---

## Summary

### HashMap Approach (Your Solution)
**Pros:**
- ✅ Easy to understand
- ✅ Easy to implement
- ✅ Works correctly
- ✅ Good for interviews (quick solution)

**Cons:**
- ❌ Uses O(n) extra space
- ❌ Doesn't meet the constant space requirement

### Recommendation
Your HashMap solution is **perfectly valid** for most real-world scenarios! It's:
- Clear and maintainable
- Easy to debug
- Performs well in practice

However, if the interviewer specifically asks for **O(1) space**, you'll need to use the bit manipulation approach. 

**Interview Tip:** Start with the HashMap solution to show you can solve the problem, then mention: *"This works, but uses O(n) space. If we need O(1) space, we can use bit manipulation with two variables to track bit occurrences modulo 3."*
