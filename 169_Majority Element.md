# Majority Element

## Problem Statement

Given an array `nums` of size `n`, return the **majority element**.

The majority element is the element that appears **more than ⌊n / 2⌋ times**. You may assume that the majority element always exists in the array.

---

## Examples

### Example 1:
```
Input: nums = [3,2,3]
Output: 3
```

### Example 2:
```
Input: nums = [2,2,1,1,1,2,2]
Output: 2
```

---

## Constraints

* `n == nums.length`
* `1 <= n <= 5 * 10^4`
* `-10^9 <= nums[i] <= 10^9`

---

## Follow-up

Could you solve the problem in **linear time** and in **O(1) space**?

---

## Solutions Analysis

### ❌ Your Approach: Dictionary (NOT OPTIMAL)

**Time:** O(n) | **Space:** O(n)

```csharp
public class Solution {
    public int MajorityElement(int[] nums) {
        Dictionary<int,int> dict = new Dictionary<int,int>();
        
         foreach(int num in nums) {
            dict[num] = dict.ContainsKey(num) ? dict[num] + 1 : 1;
             if(dict[num] > nums.Length/2) {
                return num;
            }
        }
        return -1;
    }
}
```

**Issues:**
- ❌ Uses O(n) space (violates O(1) space requirement)
- ✅ Works correctly
- ⚠️ Two passes through data

---

### ✅ Approach 1: Boyer-Moore Voting Algorithm (OPTIMAL) ⭐

**Time:** O(n) | **Space:** O(1)

```csharp
public class Solution {
    public int MajorityElement(int[] nums) {
        int candidate = nums[0];
        int count = 1;
        
        // Phase 1: Find candidate
        for (int i = 1; i < nums.Length; i++) {
            if (count == 0) {
                candidate = nums[i];
                count = 1;
            }
            else if (nums[i] == candidate) {
                count++;
            }
            else {
                count--;
            }
        }
        
        // Phase 2: Verify (optional if majority always exists)
        // count = 0;
        // foreach (int num in nums) {
        //     if (num == candidate) count++;
        // }
        // return count > nums.Length / 2 ? candidate : -1;
        
        return candidate;
    }
}
```

**Advantages:**
- ✅ O(n) time - single pass
- ✅ O(1) space - only two variables
- ✅ Meets follow-up requirements perfectly
- ✅ Most elegant solution

---

### ✅ Approach 2: Sorting

**Time:** O(n log n) | **Space:** O(1) or O(n)

```csharp
public class Solution {
    public int MajorityElement(int[] nums) {
        Array.Sort(nums);
        return nums[nums.Length / 2];
    }
}
```

**Why it works:** If an element appears more than n/2 times, it MUST be at the middle position after sorting.

**Advantages:**
- ✅ Very simple
- ✅ O(1) space (in-place sort)
- ❌ O(n log n) time (not optimal)

---

### ✅ Approach 3: Bit Manipulation

**Time:** O(32n) = O(n) | **Space:** O(1)

```csharp
public class Solution {
    public int MajorityElement(int[] nums) {
        int majority = 0;
        int n = nums.Length;
        
        // Check each bit position (32 bits for int)
        for (int i = 0; i < 32; i++) {
            int bitCount = 0;
            
            // Count how many numbers have this bit set
            foreach (int num in nums) {
                if (((num >> i) & 1) == 1) {
                    bitCount++;
                }
            }
            
            // If more than half have this bit, majority has it too
            if (bitCount > n / 2) {
                majority |= (1 << i);
            }
        }
        
        return majority;
    }
}
```

**Advantages:**
- ✅ O(n) time
- ✅ O(1) space
- ✅ Clever bit-by-bit construction
- ❌ More complex than Boyer-Moore

---

### ✅ Approach 4: Randomization

**Time:** O(∞) expected O(n) | **Space:** O(1)

```csharp
public class Solution {
    public int MajorityElement(int[] nums) {
        Random rand = new Random();
        int majorityCount = nums.Length / 2;
        
        while (true) {
            int candidate = nums[rand.Next(nums.Length)];
            int count = 0;
            
            foreach (int num in nums) {
                if (num == candidate) count++;
            }
            
            if (count > majorityCount) {
                return candidate;
            }
        }
    }
}
```

**Why it works:** Since majority element is > n/2, probability of picking it is > 50%. Expected iterations = 2.

---

## Performance Comparison

```
┌─────────────────────────┬────────────┬─────────┬──────────────┬────────────┐
│ Approach                │ Time       │ Space   │ Optimal?     │ Difficulty │
├─────────────────────────┼────────────┼─────────┼──────────────┼────────────┤
│ Dictionary (Your Sol)   │ O(n)       │ O(n)    │ ❌ No        │ Easy       │
│ Boyer-Moore Voting ⭐   │ O(n)       │ O(1)    │ ✅ Yes       │ Medium     │
│ Sorting                 │ O(n log n) │ O(1)    │ ⚠️ Time      │ Easy       │
│ Bit Manipulation        │ O(n)       │ O(1)    │ ✅ Yes       │ Hard       │
│ Randomization           │ O(n) avg   │ O(1)    │ ✅ Expected  │ Medium     │
└─────────────────────────┴────────────┴─────────┴──────────────┴────────────┘

Follow-up Requirements: O(n) time + O(1) space
✅ Boyer-Moore, Bit Manipulation, Randomization qualify
⭐ Boyer-Moore is the expected answer
```

---

## Boyer-Moore Voting Algorithm Explained

### The Genius Insight

**Key Observation:** If we pair up each majority element with a different element, there will still be majority elements left over (since it appears > n/2 times).

**Strategy:** Treat the problem like voting:
- Each occurrence of the candidate is a +1 vote
- Each different element is a -1 vote (cancellation)
- If votes reach 0, switch candidates

**Why it works:** The majority element will survive all cancellations!

### The Two Phases

#### Phase 1: Find Candidate (Required)
```
Scan array and maintain:
- candidate: current candidate for majority
- count: how many more of candidate vs others

Rules:
1. If count == 0: pick current element as new candidate
2. If current == candidate: count++
3. If current != candidate: count-- (cancellation)
```

#### Phase 2: Verify Candidate (Optional)
```
If problem guarantees majority exists: Skip this phase
Otherwise: Count occurrences and verify > n/2
```

---

## Visual Walkthrough: Boyer-Moore

### Example 1: `nums = [2,2,1,1,1,2,2]`

```
Initial State:
candidate = 2 (first element)
count = 1

═══════════════════════════════════════════════════════
Index 1: nums[1] = 2
         2 == candidate(2)? YES
         count++ → count = 2

Array: [2, 2, 1, 1, 1, 2, 2]
            ↑
Candidate: 2, Count: 2

═══════════════════════════════════════════════════════
Index 2: nums[2] = 1
         1 == candidate(2)? NO
         count-- → count = 1 (cancellation!)

Array: [2, 2, 1, 1, 1, 2, 2]
               ↑
Candidate: 2, Count: 1

Think: One 2 got "cancelled" by this 1

═══════════════════════════════════════════════════════
Index 3: nums[3] = 1
         1 == candidate(2)? NO
         count-- → count = 0 (candidate eliminated!)

Array: [2, 2, 1, 1, 1, 2, 2]
                  ↑
Candidate: 2, Count: 0

Think: All 2s so far got cancelled by 1s

═══════════════════════════════════════════════════════
Index 4: nums[4] = 1
         count == 0, so pick new candidate
         candidate = 1
         count = 1

Array: [2, 2, 1, 1, 1, 2, 2]
                     ↑
Candidate: 1, Count: 1

Think: Starting fresh with 1 as candidate

═══════════════════════════════════════════════════════
Index 5: nums[5] = 2
         2 == candidate(1)? NO
         count-- → count = 0

Array: [2, 2, 1, 1, 1, 2, 2]
                        ↑
Candidate: 1, Count: 0

Think: This 1 got cancelled by a 2

═══════════════════════════════════════════════════════
Index 6: nums[6] = 2
         count == 0, so pick new candidate
         candidate = 2
         count = 1

Array: [2, 2, 1, 1, 1, 2, 2]
                           ↑
Candidate: 2, Count: 1

═══════════════════════════════════════════════════════
Result: candidate = 2 ✓

Visualization of cancellations:
2 2 1 1 1 2 2
│ │ X X   │ │  ← X = cancelled pairs
└─┴─────────┘  ← Survivors are all 2s!
```

### Example 2: `nums = [3,2,3]`

```
Initial: candidate = 3, count = 1

Index 1: nums[1] = 2
         2 != 3
         count-- → count = 0

Index 2: nums[2] = 3
         count == 0, pick new candidate
         candidate = 3
         count = 1

Result: candidate = 3 ✓

Cancellations:
3 2 3
│ X │  ← 2 cancels one 3
└───┘  ← 3 survives!
```

---

## Why Boyer-Moore Works: Mathematical Proof

### Theorem
If an element appears more than ⌊n/2⌋ times, Boyer-Moore will find it.

### Proof

**Given:**
- Majority element M appears k times where k > n/2
- Other elements collectively appear (n - k) times where (n - k) < n/2

**Claim:** M will be the final candidate

**Proof by Contradiction:**

Assume the final candidate is NOT M.

1. Every occurrence of M either:
   - Increases count when M is candidate, OR
   - Decreases count when M is not candidate

2. Every non-M element either:
   - Increases count when it's candidate, OR
   - Decreases count when it's not candidate

3. Worst case for M: All its occurrences are used to cancel non-M elements
   - M appears k times
   - Non-M appears (n-k) times
   - After cancellation: k - (n-k) = k - n + k = 2k - n remaining

4. Since k > n/2:
   - 2k > n
   - 2k - n > 0
   - So M elements remain after all cancellations

5. Therefore, M must survive and be the final candidate!

**Contradiction!** Our assumption was wrong.

**Therefore, Boyer-Moore correctly finds the majority element.** ∎

---

## Why Sorting Works

### The Middle Element Lemma

**Claim:** If element E appears > n/2 times, then after sorting, `nums[n/2]` must be E.

**Proof:**

```
Case 1: n is odd (e.g., n=7)
Middle index = 3

If E appears > 3.5 times (at least 4 times):
┌───┬───┬───┬───┬───┬───┬───┐
│ ? │ ? │ ? │ E │ E │ E │ E │  ← E must cover middle
└───┴───┴───┴───┴───┴───┴───┘
         middle→ ↑

Case 2: n is even (e.g., n=8)
Middle index = 4

If E appears > 4 times (at least 5 times):
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ ? │ ? │ ? │ E │ E │ E │ E │ E │  ← E must cover middle
└───┴───┴───┴───┴───┴───┴───┴───┘
            middle→ ↑

In both cases, E occupies the middle position! ✓
```

**Visual Proof:**
```
Array of 7 elements, E appears 4+ times:

Scenario 1: E at start
[E E E E ? ? ?]
       ↑ middle is E

Scenario 2: E at end
[? ? ? E E E E]
       ↑ middle is E

Scenario 3: E in middle
[? E E E E E ?]
       ↑ middle is E

All scenarios → middle is always E!
```

---

## Complexity Analysis

### Boyer-Moore Detailed Analysis

```
Time Complexity: O(n)
─────────────────────────
Phase 1 (Finding candidate):
  for i = 1 to n-1:          O(n)
    3 comparisons max         O(1)
  Total: O(n)

Phase 2 (Verification - optional):
  foreach num in nums:        O(n)
    1 comparison              O(1)
  Total: O(n)

Overall: O(n) + O(n) = O(2n) = O(n)

Space Complexity: O(1)
──────────────────────────
Variables used:
  candidate: 1 int           4 bytes
  count: 1 int               4 bytes
Total: 8 bytes → O(1) ✓
```

### Your Dictionary Approach

```
Time Complexity: O(n)
─────────────────────────
Building dictionary:
  foreach num in nums:        O(n)
    Dictionary operations      O(1) average
  Total: O(n)

Finding majority:
  foreach item in dict:       O(k) where k = unique elements
    Comparison                O(1)
  Total: O(k) ≤ O(n)

Overall: O(n) + O(n) = O(2n) = O(n)

Space Complexity: O(n)
──────────────────────────
Dictionary storage:
  Worst case: n unique        n * (key + value + overhead)
  Average: ~n/2 unique        ~16n bytes
Total: O(n) ✗ Violates O(1) requirement!
```

---

## Memory Usage Comparison

```
For n = 10,000 elements:

═══════════════════════════════════════════════════════
DICTIONARY APPROACH
═══════════════════════════════════════════════════════

Input Array:          40,000 bytes (10,000 ints)
Dictionary:          ~160,000 bytes (worst case)
                     ════════════════════════════
TOTAL:               200,000 bytes
OVERHEAD:            160,000 bytes (400%)

═══════════════════════════════════════════════════════
BOYER-MOORE APPROACH
═══════════════════════════════════════════════════════

Input Array:          40,000 bytes
Variables (2):             8 bytes
                     ════
TOTAL:                40,008 bytes
OVERHEAD:                 8 bytes (0.02%)

✅ Boyer-Moore is 20,000× MORE MEMORY EFFICIENT!
```

---

## Edge Cases Handled

### All Approaches Handle These:

```
1. Single Element
   Input:  [1]
   Output: 1
   ✓ Majority is 100% (> 50%)

2. All Same Elements
   Input:  [5, 5, 5, 5, 5]
   Output: 5
   ✓ Majority is 100%

3. Minimum Majority (exactly > n/2)
   Input:  [1, 1, 1, 2, 2]
   Output: 1
   ✓ 1 appears 3 times, 3 > 5/2 = 2.5

4. Two Elements - Same
   Input:  [2, 2]
   Output: 2
   ✓ 2 appears 100%

5. Two Elements - Different (impossible)
   Input:  [1, 2]
   Problem states majority always exists
   This input violates constraints

6. Large Array with Clear Majority
   Input:  [1] * 10000 + [2] * 9999
   Output: 1
   ✓ 1 appears 10000 times > 19999/2

7. Negative Numbers
   Input:  [-1, -1, -1, 2, 2]
   Output: -1
   ✓ Works with any integer
```

---

## Common Mistakes

### ❌ Mistake 1: Not Resetting Candidate Correctly

```csharp
// WRONG
if (count == 0) {
    candidate = nums[i];
    // Forgot to set count = 1!
}
```

**Problem:** After picking new candidate, count stays 0
**Fix:** Always set `count = 1` when picking new candidate

---

### ❌ Mistake 2: Starting from Index 0 in Loop

```csharp
// WRONG - processes first element twice
int candidate = nums[0];
int count = 1;
for (int i = 0; i < nums.Length; i++) {  // Should start from 1
```

**Problem:** First element processed twice
**Fix:** Start loop from index 1: `for (int i = 1; ...)`

---

### ❌ Mistake 3: Wrong Comparison Order

```csharp
// WRONG - checks equality before count
if (nums[i] == candidate) {
    count++;
}
else if (count == 0) {  // Should check count first!
```

**Problem:** Misses candidate reset opportunity
**Fix:** Check `count == 0` first

---

### ❌ Mistake 4: Verification Without Guarantee

```csharp
// WRONG - skips verification when not guaranteed
public int MajorityElement(int[] nums) {
    // ... find candidate ...
    return candidate;  // What if no majority exists?
}
```

**Problem:** This problem guarantees majority, but general case needs verification
**Fix:** Add verification phase if majority not guaranteed

---

### ❌ Mistake 5: Using Wrong Threshold

```csharp
// WRONG
if (count >= nums.Length / 2) {  // Should be >
```

**Problem:** Majority is STRICTLY greater than n/2, not ≥
**Example:** [1,1,2,2] - both appear n/2=2 times, neither is majority

---

## Alternative Implementation: More Explicit

```csharp
public class Solution {
    public int MajorityElement(int[] nums) {
        int candidate = FindCandidate(nums);
        
        // Optional verification if majority not guaranteed
        if (ShouldVerify()) {
            return IsValidMajority(nums, candidate) ? candidate : -1;
        }
        
        return candidate;
    }
    
    private int FindCandidate(int[] nums) {
        int candidate = nums[0];
        int count = 1;
        
        for (int i = 1; i < nums.Length; i++) {
            if (count == 0) {
                candidate = nums[i];
                count = 1;
            }
            else if (nums[i] == candidate) {
                count++;
            }
            else {
                count--;
            }
        }
        
        return candidate;
    }
    
    private bool IsValidMajority(int[] nums, int candidate) {
        int count = 0;
        foreach (int num in nums) {
            if (num == candidate) count++;
        }
        return count > nums.Length / 2;
    }
    
    private bool ShouldVerify() {
        return false;  // Problem guarantees majority exists
    }
}
```

---

## Interview Tips

### Opening Statement

> "I'll use the Boyer-Moore Voting Algorithm, which is the optimal solution with O(n) time and O(1) space. The key insight is that we can pair up the majority element with different elements, and the majority will still have elements left over since it appears more than half the time. We treat it like voting with cancellations."

### Key Points to Mention

1. **The Cancellation Strategy**
   > "When we see the candidate, we increase the count. When we see a different element, we decrease it - like cancelling votes. If count reaches zero, we pick a new candidate."

2. **Why It Works**
   > "The majority element appears more than n/2 times. Even if every majority element cancels with a different element, there will still be majority elements remaining."

3. **Two Phases (if asked)**
   > "Typically there are two phases: finding the candidate and verifying it. Since this problem guarantees a majority exists, we can skip verification."

4. **Complexity Achievement**
   > "We only use two variables regardless of array size, achieving O(1) space. And we make one pass through the array for O(n) time."

### Follow-up Questions & Answers

**Q: What if there's no majority element guaranteed?**
> "Then I'd add a verification phase: count the candidate's occurrences and check if it's > n/2. This is still O(n) time and O(1) space."

**Q: Why not just use a HashMap?**
> "HashMap works but uses O(n) space. The follow-up specifically asks for O(1) space, which Boyer-Moore achieves elegantly."

**Q: What if we need to find elements appearing > n/3 times?**
> "We can extend Boyer-Moore to track two candidates for the n/3 case. But it becomes more complex - for n/k, we'd need k-1 candidates."

**Q: Can you explain why the final candidate is always correct?**
> "After all cancellations, the majority element must survive because it appears more than half the time. Even if every majority element pairs with a non-majority element, there will be majority elements left over."

**Q: What's the worst case for this algorithm?**
> "The worst case is when the majority element is at the end and keeps getting cancelled, requiring multiple candidate switches. But it's still O(n) time - we just do more count decrements."

---

## Related LeetCode Problems

| Problem | Difficulty | Similarity |
|---------|-----------|------------|
| **169. Majority Element** | Easy | This problem |
| 229. Majority Element II | Medium | Find elements > n/3 |
| 1150. Check If Majority Element | Easy | Verify majority |
| 2404. Most Frequent Even Element | Easy | Most frequent with condition |
| 451. Sort Characters By Frequency | Medium | Frequency-based sorting |
| 347. Top K Frequent Elements | Medium | K most frequent |

---

## Boyer-Moore Extensions

### Finding Elements > n/3 (Majority Element II)

```csharp
public IList<int> MajorityElementII(int[] nums) {
    // Can have at most 2 elements appearing > n/3 times
    int candidate1 = 0, candidate2 = 0;
    int count1 = 0, count2 = 0;
    
    // Phase 1: Find two candidates
    foreach (int num in nums) {
        if (num == candidate1) {
            count1++;
        }
        else if (num == candidate2) {
            count2++;
        }
        else if (count1 == 0) {
            candidate1 = num;
            count1 = 1;
        }
        else if (count2 == 0) {
            candidate2 = num;
            count2 = 1;
        }
        else {
            count1--;
            count2--;
        }
    }
    
    // Phase 2: Verify both candidates
    count1 = count2 = 0;
    foreach (int num in nums) {
        if (num == candidate1) count1++;
        else if (num == candidate2) count2++;
    }
    
    var result = new List<int>();
    if (count1 > nums.Length / 3) result.Add(candidate1);
    if (count2 > nums.Length / 3) result.Add(candidate2);
    
    return result;
}
```

---

## Summary

### Key Takeaways

1. ✅ **Boyer-Moore is optimal** - O(n) time, O(1) space
2. ✅ **Cancellation strategy** - Pairs cancel out, majority survives
3. ✅ **Two phases** - Find candidate, verify (optional if guaranteed)
4. ✅ **Elegant mathematics** - Provably correct algorithm
5. ✅ **Extensible** - Can adapt for n/3, n/4, etc.

### The Mental Model

```
Think of it as a battle:
┌─────────────────────────────────┐
│ Majority Army vs Others         │
│                                  │
│ Every soldier fights 1v1         │
│ Pairs eliminate each other       │
│                                  │
│ Since Majority > Others:         │
│ Majority soldiers will remain!   │
└─────────────────────────────────┘

That's Boyer-Moore Voting!
```

### Comparison Summary

```
┌──────────────────┬──────────┬─────────┬──────────────┬────────────┐
│ Your Solution    │ Time     │ Space   │ Follow-up?   │ Verdict    │
├──────────────────┼──────────┼─────────┼──────────────┼────────────┤
│ Dictionary       │  O(n)    │  O(n)   │ ❌ NO        │ Not optimal│
│ Boyer-Moore ⭐   │  O(n)    │  O(1)   │ ✅ YES       │ Perfect!   │
└──────────────────┴──────────┴─────────┴──────────────┴────────────┘

Only Boyer-Moore meets the O(1) space requirement!
This is THE expected solution for interviews.
```

### When to Use Each Approach

**Use Boyer-Moore when:**
- ✅ Need O(1) space
- ✅ Single pass preferred
- ✅ Want optimal solution
- ✅ **Always for this problem!**

**Use Dictionary when:**
- ⚠️ Need to track all frequencies
- ⚠️ Space is not a constraint
- ⚠️ Clarity over optimization
- ❌ **Not for this problem's follow-up**

**Use Sorting when:**
- ⚠️ Simplicity is priority
- ⚠️ O(n log n) acceptable
- ⚠️ In-place modification OK
- ❌ **Not optimal for follow-up**

This problem beautifully demonstrates how a clever algorithm (Boyer-Moore) can achieve optimal complexity where naive approaches fall short!
