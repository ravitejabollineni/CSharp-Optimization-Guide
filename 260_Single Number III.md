# Single Number III

## Problem Statement

Given an integer array `nums`, in which exactly two elements appear only once and all the other elements appear exactly twice. Find the two elements that appear only once. You can return the answer in **any order**.

You must write an algorithm that runs in **linear runtime complexity** and uses only **constant extra space**.

---

## Examples

### Example 1:
```
Input: nums = [1,2,1,3,2,5]
Output: [3,5]
Explanation: [5, 3] is also a valid answer.
```

### Example 2:
```
Input: nums = [-1,0]
Output: [-1,0]
```

### Example 3:
```
Input: nums = [0,1]
Output: [1,0]
```

---

## Constraints

* `2 <= nums.length <= 3 * 10^4`
* `-2^31 <= nums[i] <= 2^31 - 1`
* Each integer in `nums` will appear twice, only two integers will appear once

---

## Solutions Analysis

### ❌ Your Approach 1: Dictionary (NOT OPTIMAL)

**Time:** O(n) | **Space:** O(n)

```csharp
public class Solution {
    public int[] SingleNumber(int[] nums) {
        Dictionary<int,int> dict = new Dictionary<int,int>();
        
        foreach(int num in nums) {
            dict[num] = dict.ContainsKey(num) ? dict[num] + 1 : 1;
        }
        
        int[] result = new int[2];
        int j = 0;
        foreach(var item in dict) {
            if(item.Value == 1) {
                result[j] = item.Key;
                if(j == 2) break; 
                j++;
            }
        }
        return result;
    }
}
```

**Issues:**
- ❌ Uses O(n) extra space (violates constant space requirement)
- ❌ Two passes through data structure
- ⚠️ Has a bug: `if(j == 2)` should be `if(j == 1)` (index out of bounds)

---

### ❌ Your Approach 2: HashSet (NOT OPTIMAL)

**Time:** O(n) | **Space:** O(n)

```csharp
public class Solution {
    public int[] SingleNumber(int[] nums) {
        HashSet<int> hash = new HashSet<int>();
        
        foreach(int num in nums) {
            if(!hash.Add(num)) hash.Remove(num);
        }
        
        int[] result = new int[2];
        hash.CopyTo(result, 0);
        return result;
    }
}
```

**Issues:**
- ❌ Uses O(n) extra space (violates constant space requirement)
- ✅ Clever use of Add/Remove
- ✅ Cleaner than approach 1

---

### ✅ Optimal Approach: Bit Manipulation with XOR (RECOMMENDED)

**Time:** O(n) | **Space:** O(1) ⭐

```csharp
public class Solution {
    public int[] SingleNumber(int[] nums) {
        // Step 1: XOR all numbers to get xor = a ^ b
        int xor = 0;
        foreach (int num in nums) {
            xor ^= num;
        }
        
        // Step 2: Find rightmost set bit in xor
        // This bit is different between a and b
        int rightmostBit = xor & (-xor);
        
        // Step 3: Divide numbers into two groups and XOR separately
        int num1 = 0, num2 = 0;
        foreach (int num in nums) {
            if ((num & rightmostBit) == 0) {
                num1 ^= num;  // Group 1
            } else {
                num2 ^= num;  // Group 2
            }
        }
        
        return new int[] { num1, num2 };
    }
}
```

**Advantages:**
- ✅ O(n) time complexity
- ✅ O(1) space complexity (meets requirement!)
- ✅ Two passes through array
- ✅ No extra data structures
- ✅ Most elegant solution

---

## Algorithm Explanation: Bit Manipulation

### The XOR Properties We Use

**Property 1:** `a ^ a = 0` (any number XOR with itself is 0)
**Property 2:** `a ^ 0 = a` (any number XOR with 0 is itself)
**Property 3:** `a ^ b ^ a = b` (XOR is commutative and associative)

### The Three-Step Strategy

#### Step 1: XOR All Numbers

```
Array: [1, 2, 1, 3, 2, 5]

XOR all numbers:
1 ^ 2 ^ 1 ^ 3 ^ 2 ^ 5

Rearrange (XOR is commutative):
(1 ^ 1) ^ (2 ^ 2) ^ 3 ^ 5
   0    ^    0    ^ 3 ^ 5
         0         ^ 3 ^ 5
                  3 ^ 5

Result: xor = 3 ^ 5 = 6 (in binary: 0110)
```

**Key Insight:** All paired numbers cancel out, leaving `a ^ b` where a and b are the two unique numbers.

#### Step 2: Find a Bit Where a and b Differ

```
xor = 3 ^ 5 = 6 (binary: 0110)

This means 3 and 5 differ at positions where there's a 1.

Find rightmost set bit:
rightmostBit = xor & (-xor)

How it works:
xor =  0110
-xor = 1010 (two's complement)
&    = 0010 (keeps only rightmost 1)

rightmostBit = 2 (binary: 0010)
```

**Why rightmost?** Any bit where they differ works, but rightmost is easiest to compute.

#### Step 3: Partition and XOR

```
Use rightmostBit to divide numbers into two groups:

Group 1: (num & rightmostBit) == 0
  - Numbers where bit position 1 is 0
  
Group 2: (num & rightmostBit) != 0
  - Numbers where bit position 1 is 1

Array: [1, 2, 1, 3, 2, 5]

Check each number:
1 (binary: 001) & 0010 = 0000 → Group 1
2 (binary: 010) & 0010 = 0010 → Group 2
1 (binary: 001) & 0010 = 0000 → Group 1
3 (binary: 011) & 0010 = 0010 → Group 2
2 (binary: 010) & 0010 = 0010 → Group 2
5 (binary: 101) & 0010 = 0000 → Group 1

Group 1: [1, 1, 5] → XOR = 1 ^ 1 ^ 5 = 5
Group 2: [2, 3, 2] → XOR = 2 ^ 3 ^ 2 = 3

Result: [5, 3] ✓
```

**Why This Works:**
- One unique number goes to Group 1, the other to Group 2
- All paired numbers: both copies go to the same group and cancel out
- Each group's XOR gives us one unique number!

---

## Visual Walkthrough with Binary

### Example: `nums = [1,2,1,3,2,5]`

```
═══════════════════════════════════════════════════════
STEP 1: XOR All Numbers
═══════════════════════════════════════════════════════

Numbers in binary:
1 → 001
2 → 010
1 → 001
3 → 011
2 → 010
5 → 101

XOR Process:
    001  (1)
  ^ 010  (2)
  ------
    011
  ^ 001  (1)
  ------
    010
  ^ 011  (3)
  ------
    001
  ^ 010  (2)
  ------
    011
  ^ 101  (5)
  ------
    110  (6) ← xor = 6

═══════════════════════════════════════════════════════
STEP 2: Find Rightmost Set Bit
═══════════════════════════════════════════════════════

xor = 6 (binary: 0110)

Two's complement of 6:
Original:  0110
Invert:    1001
Add 1:     1010  ← -6

xor & (-xor):
  0110  (6)
& 1010  (-6)
------
  0010  (2) ← rightmostBit = 2

This is bit position 1 (counting from right, 0-indexed)

═══════════════════════════════════════════════════════
STEP 3: Partition into Two Groups
═══════════════════════════════════════════════════════

Check bit position 1 for each number:

Number | Binary | & 0010 | Group
-------|--------|--------|-------
  1    |  001   |  000   |   1
  2    |  010   |  010   |   2
  1    |  001   |  000   |   1
  3    |  011   |  010   |   2
  2    |  010   |  010   |   2
  5    |  101   |  000   |   1

Group 1 (bit 1 = 0): [1, 1, 5]
  XOR: 1 ^ 1 ^ 5 = 0 ^ 5 = 5

Group 2 (bit 1 = 1): [2, 3, 2]
  XOR: 2 ^ 3 ^ 2 = 3

═══════════════════════════════════════════════════════
RESULT: [5, 3] ✓
═══════════════════════════════════════════════════════
```

---

## Understanding `-xor` (Two's Complement)

### What is Two's Complement?

To negate a number in binary:
1. **Invert all bits** (0→1, 1→0)
2. **Add 1**

### Why Does `xor & (-xor)` Give Rightmost Bit?

```
Example: xor = 6 (0110)

Step 1: Invert bits
0110 → 1001

Step 2: Add 1
1001 + 1 = 1010

Now AND them:
  0110  (original)
& 1010  (negated)
------
  0010  (rightmost 1 isolated!)

Why? In two's complement:
- All bits to the right of rightmost 1 are 0 in both
- Rightmost 1 stays as 1 in both
- All bits to the left are flipped (so AND gives 0)
```

### Visual Proof

```
Original: ...ABC1000  (where ABC are any bits)
Invert:   ...XYZ0111  (where XYZ = NOT ABC)
Add 1:    ...XYZ1000

AND:
  ...ABC1000
& ...XYZ1000
------------
  ...0001000  (only rightmost 1 survives!)
```

---

## Complexity Analysis

### Performance Comparison

```
┌─────────────────────┬──────────┬─────────┬──────────────┬────────────┐
│ Approach            │ Time     │ Space   │ Meets Req?   │ Optimal?   │
├─────────────────────┼──────────┼─────────┼──────────────┼────────────┤
│ Dictionary          │  O(n)    │  O(n)   │ ❌ No        │ ❌ No      │
│ HashSet             │  O(n)    │  O(n)   │ ❌ No        │ ❌ No      │
│ Bit Manipulation    │  O(n)    │  O(1)   │ ✅ Yes       │ ✅ Yes ⭐  │
└─────────────────────┴──────────┴─────────┴──────────────┴────────────┘

Problem Requirement: O(n) time, O(1) space
Only Bit Manipulation meets both requirements!
```

### Detailed Time Analysis

```
Bit Manipulation Approach:

Pass 1: XOR all numbers
  for (int num in nums)  → O(n)
    xor ^= num;          → O(1)
  Total: O(n)

Pass 2: Find rightmost bit
  xor & (-xor)           → O(1)

Pass 3: Partition and XOR
  for (int num in nums)  → O(n)
    Group and XOR        → O(1)
  Total: O(n)

TOTAL TIME: O(n) + O(1) + O(n) = O(2n) = O(n) ✓
```

### Space Analysis

```
Bit Manipulation:
  xor variable:         4 bytes
  rightmostBit:         4 bytes
  num1, num2:           8 bytes
  result array:         8 bytes
  
Total: ~24 bytes regardless of input size → O(1) ✓

Dictionary/HashSet:
  Worst case stores n/2 unique numbers
  Each entry: ~16 bytes (key + overhead)
  
Total: ~8n bytes → O(n) ✗
```

---

## Memory Usage Visualization

```
Input Size: n = 10,000 elements

═══════════════════════════════════════════════════════
DICTIONARY APPROACH
═══════════════════════════════════════════════════════

Input Array:          40,000 bytes (10,000 ints)
Dictionary:          ~80,000 bytes (storing up to 5,000 unique)
                     ════════════════════════════════════

TOTAL MEMORY:        120,000 bytes
WASTED:               80,000 bytes (200% overhead!)

═══════════════════════════════════════════════════════
HASHSET APPROACH
═══════════════════════════════════════════════════════

Input Array:          40,000 bytes
HashSet:             ~80,000 bytes (storing up to 5,000)
                     ════════════════════════════════════

TOTAL MEMORY:        120,000 bytes
WASTED:               80,000 bytes (200% overhead!)

═══════════════════════════════════════════════════════
BIT MANIPULATION APPROACH
═══════════════════════════════════════════════════════

Input Array:          40,000 bytes
Variables (5):            24 bytes
Result Array:              8 bytes
                     ════

TOTAL MEMORY:         40,032 bytes
WASTED:                   32 bytes (0.08% overhead!)

✅ 2,998× MORE EFFICIENT than Dictionary/HashSet!
```

---

## Edge Cases Handled

### 1. Two Elements Only
```
Input:  [-1, 0]
Step 1: xor = -1 ^ 0 = -1
Step 2: rightmost bit found
Step 3: Partition and XOR
Output: [-1, 0] ✓
```

### 2. Negative Numbers
```
Input:  [-1, -1, 1, 2, 1, 3, 2]
XOR handles negative numbers correctly (two's complement)
Output: [-1, 3] or [3, -1] ✓
```

### 3. Zero as Unique Number
```
Input:  [0, 1, 0, 2]
Step 1: xor = 1 ^ 2 = 3
Output: [1, 2] ✓
```

### 4. Large Numbers
```
Input:  [2147483647, -2147483648, 2147483647, 100]
XOR works with full integer range
Output: [-2147483648, 100] ✓
```

### 5. Many Duplicates
```
Input:  [1, 1, 2, 2, 3, 3, 4, 4, 5, 6]
All duplicates cancel out
Output: [5, 6] ✓
```

---

## Common Mistakes

### ❌ Mistake 1: Not Understanding Two's Complement

```csharp
// WRONG - trying to find rightmost bit differently
int rightmostBit = 1;
while ((xor & rightmostBit) == 0) {
    rightmostBit <<= 1;
}
```

**Problem:** More complex, more code, same result
**Fix:** Use `xor & (-xor)` - elegant and efficient

---

### ❌ Mistake 2: Wrong Bit Position Check

```csharp
// WRONG
if ((num & rightmostBit) == 1) {  // Should be != 0
```

**Problem:** If rightmostBit is 2 (0010), the result won't be 1!
**Fix:** Check `!= 0` or `> 0`, not `== 1`

---

### ❌ Mistake 3: Not Handling Negatives in Two's Complement

```csharp
// WRONG - assuming simple bit flip
int rightmostBit = ~xor;  // This is NOT two's complement!
```

**Problem:** `~xor` is one's complement (just flips bits)
**Fix:** Use `-xor` which is two's complement (flip + add 1)

---

### ❌ Mistake 4: Using Dictionary When Not Needed

```csharp
// WRONG - violates O(1) space requirement
Dictionary<int, int> dict = new Dictionary<int, int>();
```

**Problem:** Problem explicitly requires constant space
**Fix:** Use bit manipulation approach

---

### ❌ Mistake 5: Array Index Bug (Your Code)

```csharp
// WRONG (from your approach 1)
if(j == 2) break;  // j can only be 0 or 1!
```

**Problem:** Array size is 2, valid indices are 0 and 1
**Fix:** `if(j == 1) break;` or better: `if(j >= 2) break;`

---

## Why Your Approaches Don't Meet Requirements

### Problem Requirements
> "You must write an algorithm that runs in linear runtime complexity and uses only constant extra space."

### Analysis

**Your Dictionary Approach:**
```
✅ Linear time: O(n)
❌ Constant space: O(n) - FAILS REQUIREMENT

Reason: Dictionary stores up to n/2 key-value pairs
```

**Your HashSet Approach:**
```
✅ Linear time: O(n)
❌ Constant space: O(n) - FAILS REQUIREMENT

Reason: HashSet stores up to n/2 elements
```

**Bit Manipulation:**
```
✅ Linear time: O(n)
✅ Constant space: O(1) - MEETS REQUIREMENT

Reason: Only uses a few integer variables
```

---

## Alternative Implementations

### Using LINQ and Cleaner Syntax

```csharp
public class Solution {
    public int[] SingleNumber(int[] nums) {
        // Step 1: Get XOR of both unique numbers
        int xor = nums.Aggregate(0, (acc, num) => acc ^ num);
        
        // Step 2: Find rightmost set bit
        int rightmostBit = xor & -xor;
        
        // Step 3: Partition and find each unique number
        int num1 = nums.Where(n => (n & rightmostBit) == 0)
                       .Aggregate(0, (acc, n) => acc ^ n);
        
        int num2 = nums.Where(n => (n & rightmostBit) != 0)
                       .Aggregate(0, (acc, n) => acc ^ n);
        
        return new int[] { num1, num2 };
    }
}
```

**Pros:** More functional, cleaner
**Cons:** Multiple passes through array, less efficient

---

### With Comments for Interview

```csharp
public class Solution {
    public int[] SingleNumber(int[] nums) {
        // XOR all numbers: pairs cancel out, left with a^b
        int xor = 0;
        foreach (int num in nums) {
            xor ^= num;
        }
        
        // Find any bit where a and b differ (rightmost is easiest)
        // Two's complement trick: keeps only rightmost set bit
        int differentBit = xor & (-xor);
        
        // Partition into two groups based on this bit
        // Each group will have one unique number + pairs
        int a = 0, b = 0;
        foreach (int num in nums) {
            if ((num & differentBit) == 0) {
                a ^= num;  // Group 1: pairs cancel, leaving first unique
            } else {
                b ^= num;  // Group 2: pairs cancel, leaving second unique
            }
        }
        
        return new int[] { a, b };
    }
}
```

---

## Interview Tips

### Opening Statement

> "This problem requires O(1) space, which rules out HashSet or Dictionary approaches. I'll use bit manipulation with XOR properties. The key insight is that XOR-ing all numbers gives us a^b, then we can find a bit where they differ to partition the array into two groups, each containing one unique number."

### Key Points to Mention

1. **XOR Properties**
   > "XOR has the property that a^a=0 and a^0=a, so pairs cancel out when we XOR everything."

2. **Two's Complement Trick**
   > "Using xor & (-xor) isolates the rightmost set bit, which tells us a position where the two unique numbers differ."

3. **Partitioning Strategy**
   > "We use this bit to divide all numbers into two groups. The unique numbers go to different groups, and pairs stay together. XOR-ing each group gives us the two unique numbers."

4. **Constant Space Achievement**
   > "We only use a few integer variables regardless of input size, achieving O(1) space as required."

### Follow-up Questions & Answers

**Q: What if there were three unique numbers?**
> "This specific approach works for exactly two unique numbers. For three, we'd need a different strategy - possibly multiple passes with different bit positions, or accept O(n) space with a set."

**Q: Why not just use a HashSet?**
> "The problem explicitly requires O(1) constant space. HashSet would use O(n) space in the worst case, violating this requirement. The bit manipulation approach is the only way to meet both time and space constraints."

**Q: What if all numbers were the same except two?**
> "The algorithm still works perfectly. For example, [5,5,5,5,5,5,3,7] - all the 5s cancel out, leaving 3^7."

**Q: Can you explain two's complement more?**
> "Two's complement is how negative numbers are stored. To negate a number: flip all bits and add 1. When we AND xor with -xor, it isolates the rightmost 1 bit because all bits to the right are 0 in both, and all bits to the left are inverted so they cancel out."

---

## Related LeetCode Problems

| Problem | Difficulty | Similarity |
|---------|-----------|------------|
| **260. Single Number III** | Medium | This problem |
| 136. Single Number | Easy | One unique number (simpler XOR) |
| 137. Single Number II | Medium | One unique, others appear 3× |
| 268. Missing Number | Easy | Uses XOR property |
| 287. Find the Duplicate Number | Medium | Bit manipulation variation |
| 389. Find the Difference | Easy | XOR application |
| 421. Maximum XOR | Medium | Advanced XOR usage |

---

## XOR Cheat Sheet

### Essential XOR Properties

```
1. a ^ a = 0           (self-cancel)
2. a ^ 0 = a           (identity)
3. a ^ b = b ^ a       (commutative)
4. a ^ b ^ c = a ^ (b ^ c)  (associative)
5. a ^ b ^ a = b       (cancellation)
```

### Common XOR Patterns

```
Problem Type          | Pattern
---------------------|----------------------------
Find single unique   | XOR all → result is unique
Find two uniques     | XOR all, then partition
Swap without temp    | a ^= b; b ^= a; a ^= b;
Check if same        | a ^ b == 0 means a == b
Toggle bit           | num ^= (1 << position)
```

---

## Why Bit Manipulation is Beautiful

### The Mathematics Behind It

```
Given: [a, a, b, b, c, d]
where a, b appear twice; c, d appear once

XOR all:
a ^ a ^ b ^ b ^ c ^ d
= (a ^ a) ^ (b ^ b) ^ c ^ d    (associative)
= 0 ^ 0 ^ c ^ d                (self-cancel)
= c ^ d                        (identity)

Now we have c ^ d, which contains all the differences!

Key insight: If bit i is set in c^d, then:
- c has 0 at position i and d has 1, OR
- c has 1 at position i and d has 0

So we can use ANY set bit to partition!
```

### Why It's Efficient

1. **No allocations**: No heap memory used
2. **Cache-friendly**: Linear scan through array
3. **Branch-free**: XOR is a simple CPU instruction
4. **Constant operations**: Bitwise ops are O(1)

---

## Summary

### Key Takeaways

1. ✅ **XOR cancels pairs** - Foundation of the solution
2. ✅ **Two's complement trick** - Isolates rightmost bit
3. ✅ **Partitioning strategy** - Divides array into two groups
4. ✅ **O(1) space** - Only solution that meets requirement
5. ✅ **Mathematical elegance** - Pure bit manipulation

### The Three-Step Algorithm

```
Step 1: XOR all → Get a ^ b
        ↓
Step 2: Find different bit → Partition criteria
        ↓
Step 3: Partition & XOR → Get a and b separately
```

### Comparison Summary

```
┌─────────────────┬──────────┬─────────┬──────────────┐
│ Your Solutions  │ Time     │ Space   │ Verdict      │
├─────────────────┼──────────┼─────────┼──────────────┤
│ Dictionary      │  O(n)    │  O(n)   │ ❌ Fails     │
│ HashSet         │  O(n)    │  O(n)   │ ❌ Fails     │
│ Bit Manipulation│  O(n)    │  O(1)   │ ✅ Perfect!  │
└─────────────────┴──────────┴─────────┴──────────────┘

Only bit manipulation meets the O(1) space requirement!
This is the ONLY acceptable solution for this problem.
```

### When to Use This Pattern

Use XOR-based bit manipulation when:
- ✅ Finding unique elements in arrays with pairs
- ✅ O(1) space is required
- ✅ Numbers can be treated as bit patterns
- ✅ Need elegant mathematical solution

This problem beautifully demonstrates the power of bit manipulation and is a classic example of thinking beyond traditional data structures!
