# 75. Sort Colors (Dutch National Flag Problem)

## Problem Statement

Given an array `nums` with `n` objects colored red, white, or blue, sort them **in-place** so that objects of the same color are adjacent, with the colors in the order red, white, and blue.

We will use the integers `0`, `1`, and `2` to represent the color red, white, and blue, respectively.

You must solve this problem **without using the library's sort function**.

---

## Examples

### Example 1:
```
Input: nums = [2,0,2,1,1,0]
Output: [0,0,1,1,2,2]
```

### Example 2:
```
Input: nums = [2,0,1]
Output: [0,1,2]
```

---

## Constraints

* `n == nums.length`
* `1 <= n <= 300`
* `nums[i]` is either `0`, `1`, or `2`

---

## Follow-up

Could you come up with a **one-pass algorithm** using only **constant extra space**?

---

## Solution: Dutch National Flag Algorithm

### ✅ Optimal Approach: Three Pointers (One Pass)

**Time:** O(n) | **Space:** O(1)

```csharp
public class Solution {
    public void SortColors(int[] nums) {
        int low = 0;           // Boundary for 0s
        int mid = 0;           // Current element being examined
        int high = nums.Length - 1;  // Boundary for 2s
        
        while (mid <= high) {
            switch (nums[mid]) {
                case 0:  // Red - move to front
                    int temp = nums[mid];
                    nums[mid] = nums[low];
                    nums[low] = temp;
                    mid++;
                    low++;
                    break;
                    
                case 1:  // White - already in correct region
                    mid++;
                    break;
                    
                case 2:  // Blue - move to back
                    int temp1 = nums[mid];
                    nums[mid] = nums[high];
                    nums[high] = temp1;
                    high--;
                    // Note: Don't increment mid here!
                    break;
            }
        }
    }
}
```

---

## Algorithm Explanation: Dutch National Flag

### History & Origin

This algorithm was designed by **Edsger W. Dijkstra** and is called the **Dutch National Flag problem** because it resembles sorting the three colors of the Dutch flag (red, white, blue).

### Core Concept: Three Regions

The algorithm maintains **three regions** in the array:

```
[0...low-1]   → All 0s (Red)
[low...mid-1] → All 1s (White)
[mid...high]  → Unknown (Being processed)
[high+1...n]  → All 2s (Blue)
```

**Visual Representation:**
```
┌─────────┬─────────┬───────────┬─────────┐
│   0s    │   1s    │ Unknown   │   2s    │
│  (Red)  │ (White) │ (Process) │ (Blue)  │
└─────────┴─────────┴───────────┴─────────┘
    ↑         ↑          ↑          ↑
   low       mid        mid        high
 (start)   (current)              (end)
```

### The Three Pointers

1. **`low`**: Marks the boundary where next 0 should be placed
2. **`mid`**: Current element being examined
3. **`high`**: Marks the boundary where next 2 should be placed

### The Three Cases

**Case 1: nums[mid] == 0 (Red)**
- Swap with element at `low`
- Move both `low` and `mid` forward
- Why move mid? We know nums[low] was 1 (already processed)

**Case 2: nums[mid] == 1 (White)**
- Already in correct region
- Just move `mid` forward
- Nothing to swap

**Case 3: nums[mid] == 2 (Blue)**
- Swap with element at `high`
- Move `high` backward
- **DON'T move mid!** We need to check what we swapped in

---

## Visual Walkthrough

### Example: `nums = [2,0,2,1,1,0]`

```
Initial State:
Array: [2, 0, 2, 1, 1, 0]
        ↑              ↑
       low,mid        high

Regions:
[0] - no 0s yet
[1] - no 1s yet
[2,0,2,1,1,0] - all unknown
[n+1] - no 2s yet

═══════════════════════════════════════════════════════

Step 1: mid=0, nums[mid]=2 (Blue)
Action: Swap nums[mid] with nums[high]
        Swap nums[0] with nums[5]
        
Before: [2, 0, 2, 1, 1, 0]
         ↑              ↑
        mid           high

After:  [0, 0, 2, 1, 1, 2]
         ↑           ↑
        mid         high
        
        high-- (high = 4)
        DON'T move mid! Need to check new value
        
Regions: [0] [1] [0,0,2,1,1] [2]

═══════════════════════════════════════════════════════

Step 2: mid=0, nums[mid]=0 (Red)
Action: Swap nums[mid] with nums[low]
        (They're the same position, so no change)
        
Array:  [0, 0, 2, 1, 1, 2]
         ↑           ↑
       low,mid      high
        
        low++ (low = 1)
        mid++ (mid = 1)
        
Regions: [0] [1] [0,2,1,1] [2]

═══════════════════════════════════════════════════════

Step 3: mid=1, nums[mid]=0 (Red)
Action: Swap nums[mid] with nums[low]
        Swap nums[1] with nums[1]
        
Array:  [0, 0, 2, 1, 1, 2]
            ↑        ↑
          low,mid   high
        
        low++ (low = 2)
        mid++ (mid = 2)
        
Regions: [0,0] [1] [2,1,1] [2]

═══════════════════════════════════════════════════════

Step 4: mid=2, nums[mid]=2 (Blue)
Action: Swap nums[mid] with nums[high]
        Swap nums[2] with nums[4]
        
Before: [0, 0, 2, 1, 1, 2]
               ↑     ↑
              mid   high

After:  [0, 0, 1, 1, 2, 2]
               ↑  ↑
              mid high
        
        high-- (high = 3)
        DON'T move mid!
        
Regions: [0,0] [1] [1,1,2] [2,2]

═══════════════════════════════════════════════════════

Step 5: mid=2, nums[mid]=1 (White)
Action: Already in correct position
        
Array:  [0, 0, 1, 1, 2, 2]
               ↑  ↑
              mid high
        
        mid++ (mid = 3)
        
Regions: [0,0] [1,1] [1,2] [2,2]

═══════════════════════════════════════════════════════

Step 6: mid=3, nums[mid]=1 (White)
Action: Already in correct position
        
Array:  [0, 0, 1, 1, 2, 2]
                  ↑
                mid,high
        
        mid++ (mid = 4)
        
Regions: [0,0] [1,1,1] [2] [2,2]

═══════════════════════════════════════════════════════

Exit: mid > high (4 > 3)

Final Result: [0, 0, 1, 1, 2, 2] ✓

Regions: [0,0] [1,1,1] [] [2,2,2]
All sorted!
```

---

## Why Don't We Increment `mid` When Swapping with `high`?

### Critical Insight

```csharp
case 2:  // nums[mid] == 2
    swap(nums[mid], nums[high]);
    high--;
    // ❌ DON'T do mid++ here!
```

**Reason:** When we swap `nums[mid]` with `nums[high]`, we don't know what value came from `nums[high]`!

**Example:**
```
Array: [1, 2, 0, 2, 1]
           ↑     ↑
          mid   high

nums[mid] = 2, so swap with nums[high]

After:  [1, 1, 0, 2, 2]
           ↑     ↑
          mid   high-1

The new nums[mid] = 1 (came from position high)
We MUST check this 1 in the next iteration!
If we did mid++, we'd skip it!
```

**But when swapping with `low`, we CAN increment:**
```csharp
case 0:  // nums[mid] == 0
    swap(nums[mid], nums[low]);
    low++;
    mid++;  // ✅ Safe to increment!
```

**Reason:** The region `[low...mid-1]` contains only 1s (already processed). So we know `nums[low]` is either:
- A 1 (if low < mid), which we've already processed
- The same position as mid (if low == mid), so the swap is harmless

---

## Complexity Analysis

### Time Complexity: **O(n)**

```
Single pass through the array:
- Each element is visited at most once by mid pointer
- mid only moves forward (except when processing 2s)
- Total operations: exactly n comparisons

Worst case: All elements are 2s
- mid stays at 0
- high moves from n-1 to 0
- n swaps performed
- Still O(n)

Best case: Already sorted [0,0,1,1,2,2]
- mid moves from 0 to n
- No swaps needed
- O(n)
```

### Space Complexity: **O(1)**

```
Only uses three pointer variables:
- low: 4 bytes
- mid: 4 bytes
- high: 4 bytes
- temp variables: 4 bytes

Total: ~16 bytes regardless of input size
Perfect O(1) constant space!
```

---

## Comparison with Other Approaches

### Approach 1: Library Sort (NOT ALLOWED)

```csharp
// ❌ Problem explicitly forbids this!
Array.Sort(nums);
```

**Time:** O(n log n) | **Space:** O(1) or O(n)
**Status:** Forbidden by problem statement

---

### Approach 2: Counting Sort (Two Pass)

```csharp
public void SortColors(int[] nums) {
    int count0 = 0, count1 = 0, count2 = 0;
    
    // First pass: Count occurrences
    foreach (int num in nums) {
        if (num == 0) count0++;
        else if (num == 1) count1++;
        else count2++;
    }
    
    // Second pass: Overwrite array
    int i = 0;
    while (count0-- > 0) nums[i++] = 0;
    while (count1-- > 0) nums[i++] = 1;
    while (count2-- > 0) nums[i++] = 2;
}
```

**Time:** O(n) | **Space:** O(1)
**Status:** ✅ Valid but TWO passes (not optimal for follow-up)

---

### Approach 3: Dutch National Flag (ONE PASS) ⭐

```csharp
// Your solution - OPTIMAL!
```

**Time:** O(n) | **Space:** O(1)
**Status:** ✅ Perfect! Answers the follow-up question

---

## Performance Comparison

```
┌────────────────────────┬──────────┬─────────┬──────────┬────────────┐
│ Approach               │ Time     │ Space   │ Passes   │ Allowed?   │
├────────────────────────┼──────────┼─────────┼──────────┼────────────┤
│ Library Sort           │ O(nlogn) │  O(?)   │   1      │ ❌ No      │
│ Counting Sort          │  O(n)    │  O(1)   │   2      │ ⚠️ Yes     │
│ Dutch National Flag    │  O(n)    │  O(1)   │   1      │ ✅ Yes ⭐  │
└────────────────────────┴──────────┴─────────┴──────────┴────────────┘

Winner: Dutch National Flag
- Answers the follow-up question perfectly
- Single pass with constant space
- Most elegant solution
```

---

## Edge Cases Handled

### 1. All Same Color
```
Input:  [0, 0, 0, 0]
Output: [0, 0, 0, 0]
✓ low moves forward, mid moves forward
```

### 2. Already Sorted
```
Input:  [0, 1, 2]
Output: [0, 1, 2]
✓ Minimal swaps, optimal performance
```

### 3. Reverse Sorted
```
Input:  [2, 2, 1, 0, 0]
Output: [0, 0, 1, 2, 2]
✓ Maximum swaps, still O(n)
```

### 4. Single Element
```
Input:  [1]
Output: [1]
✓ mid <= high initially false, exits immediately
```

### 5. Two Elements - Same
```
Input:  [1, 1]
Output: [1, 1]
✓ Both mid pointers move forward
```

### 6. Two Elements - Need Swap
```
Input:  [2, 0]
Output: [0, 2]
✓ Swap performed correctly
```

### 7. Only Two Colors
```
Input:  [0, 2, 0, 2, 0, 2]
Output: [0, 0, 0, 2, 2, 2]
✓ No 1s, works perfectly
```

### 8. Alternating Pattern
```
Input:  [2, 0, 2, 0, 2, 0]
Output: [0, 0, 0, 2, 2, 2]
✓ Maximum number of swaps
```

---

## Common Mistakes to Avoid

### ❌ Mistake 1: Incrementing mid After Swapping with high

```csharp
// WRONG!
case 2:
    swap(nums[mid], nums[high]);
    high--;
    mid++;  // ❌ BUG! Skips the swapped element
    break;
```

**Problem:** You don't know what value was at `nums[high]`!

**Fix:** Don't increment mid when swapping with high

---

### ❌ Mistake 2: Wrong Loop Condition

```csharp
// WRONG!
while (mid < high) {  // Should be <=
```

**Problem:** Misses the element when mid == high

**Example:**
```
[0, 1, 2]
After processing 0 and 1:
mid = 2, high = 2
mid < high is false, exits without processing last element!
```

**Fix:** Use `while (mid <= high)`

---

### ❌ Mistake 3: Wrong Initial Values

```csharp
// WRONG!
int low = 0;
int mid = 1;  // Should be 0
int high = nums.Length - 1;
```

**Problem:** Skips first element!

**Fix:** Start mid at 0

---

### ❌ Mistake 4: Swapping Without Temp Variable

```csharp
// WRONG! Only works for XOR of different values
nums[mid] = nums[low];
nums[low] = nums[mid];  // Both become nums[low]!
```

**Fix:** Always use a temp variable or tuple swap

---

### ❌ Mistake 5: Not Handling When low == mid

```csharp
// Wrong assumption:
// "When swapping 0, low and mid must be different"

// Actually: They can be the same!
[0, 1, 2]
 ↑
low,mid  ← Same position is valid
```

**Fix:** The swap works correctly even when low == mid (swaps element with itself)

---

## Alternative Implementation: Using Tuple Swap

```csharp
public void SortColors(int[] nums) {
    int low = 0, mid = 0, high = nums.Length - 1;
    
    while (mid <= high) {
        if (nums[mid] == 0) {
            // Swap using tuple
            (nums[low], nums[mid]) = (nums[mid], nums[low]);
            low++;
            mid++;
        }
        else if (nums[mid] == 1) {
            mid++;
        }
        else {  // nums[mid] == 2
            (nums[mid], nums[high]) = (nums[high], nums[mid]);
            high--;
        }
    }
}
```

**Pros:** Cleaner syntax with C# 7.0+ tuples
**Cons:** Same performance, just different style

---

## Alternative Implementation: Using Helper Method

```csharp
public void SortColors(int[] nums) {
    int low = 0, mid = 0, high = nums.Length - 1;
    
    while (mid <= high) {
        if (nums[mid] == 0) {
            Swap(nums, low++, mid++);
        }
        else if (nums[mid] == 1) {
            mid++;
        }
        else {
            Swap(nums, mid, high--);
        }
    }
}

private void Swap(int[] nums, int i, int j) {
    int temp = nums[i];
    nums[i] = nums[j];
    nums[j] = temp;
}
```

**Pros:** More readable, reusable swap function
**Cons:** Extra function call (minimal overhead)

---

## Interview Tips

### Opening Statement

> "This is the famous Dutch National Flag problem, designed by Dijkstra. I'll use three pointers to partition the array in a single pass: one to track where 0s should go (low), one for the current element (mid), and one for where 2s should go (high). This achieves O(n) time with O(1) space in one pass."

### Key Points to Mention

1. **The Three Regions**
   > "I'm maintaining three regions: processed 0s, processed 1s, and processed 2s, with an unknown region in the middle that shrinks as mid moves forward."

2. **Why mid Doesn't Increment for Case 2**
   > "When I swap with high, I don't increment mid because I don't know what value came from that position - it could be 0, 1, or 2. I need to examine it in the next iteration."

3. **One Pass Achievement**
   > "This answers the follow-up question perfectly - it's a single pass through the array with constant extra space."

4. **Loop Condition**
   > "I use mid <= high, not mid < high, because when mid equals high, there's still one element to process."

### Follow-up Questions & Answers

**Q: What if there were 4 colors (0, 1, 2, 3)?**
> "The Dutch National Flag algorithm specifically works for 3 values. For 4+ values, I'd use counting sort (two passes) or quicksort-style partitioning (more complex with multiple pivots)."

**Q: Can you do it in one pass with 4 colors?**
> "It becomes significantly more complex and isn't as elegant. For practical purposes, counting sort with O(k) space where k is the number of colors is simpler and still O(n) time."

**Q: What's the maximum number of swaps needed?**
> "Worst case is O(n) swaps, for example when the array is [2,2,2,0,0,0] - every element needs to move. But we still only make one pass through the array."

**Q: Could we start from both ends?**
> "That's essentially what we're doing! The high pointer starts at the end and moves left, while low starts at the beginning and moves right. Mid handles the scanning."

---

## Proof of Correctness

### Invariants (Always True During Execution)

1. **Invariant 1:** All elements in `[0...low-1]` are 0s
2. **Invariant 2:** All elements in `[low...mid-1]` are 1s
3. **Invariant 3:** All elements in `[high+1...n-1]` are 2s
4. **Invariant 4:** Elements in `[mid...high]` are unknown

### Loop Termination

- Loop continues while `mid <= high`
- Each iteration either:
  - Increments mid (reducing unknown region)
  - Decrements high (reducing unknown region)
- Unknown region shrinks until `mid > high`
- Therefore, loop terminates in at most n iterations

### Correctness

- When loop exits: `mid > high`
- This means the unknown region is empty
- By invariants, all elements are in their correct regions
- Array is sorted: [all 0s][all 1s][all 2s]

**Therefore, the algorithm is correct.** ∎

---

## Related LeetCode Problems

| Problem | Difficulty | Similarity |
|---------|-----------|------------|
| **75. Sort Colors** | Medium | This problem |
| 26. Remove Duplicates from Sorted Array | Easy | Two pointer partitioning |
| 27. Remove Element | Easy | Two pointer technique |
| 283. Move Zeroes | Easy | Similar partitioning |
| 905. Sort Array By Parity | Easy | Two-way partitioning |
| 922. Sort Array By Parity II | Easy | Two-way with indices |
| 148. Sort List | Medium | Sorting with constraints |

---

## Practice Variations

1. **Four Colors**: Extend to sort [0,1,2,3]
2. **Sort by Parity**: Separate even and odd numbers
3. **Partition Around Pivot**: QuickSort partition step
4. **Multiple Criteria**: Sort by two attributes
5. **In-Place Merge**: Combine two sorted regions

---

## Why This Problem is Important

### Teaches Key Concepts

1. ✅ **Three-pointer technique** - Extension of two pointers
2. ✅ **In-place partitioning** - Core of QuickSort
3. ✅ **Invariant maintenance** - Proving correctness
4. ✅ **One-pass algorithms** - Optimization thinking
5. ✅ **Edge case handling** - When to move pointers

### Real-World Applications

- **QuickSort partitioning** - Same technique
- **Data segregation** - Separating categories
- **Memory management** - Organizing memory regions
- **Network packet routing** - Priority sorting
- **Database query optimization** - Partition pruning

---

## Summary

### Key Takeaways

1. ✅ **Dutch National Flag** is the optimal solution
2. ✅ **Three pointers** maintain three regions
3. ✅ **Don't increment mid after swapping with high** - Must examine swapped value
4. ✅ **Use `mid <= high`** as loop condition
5. ✅ **One pass, O(1) space** - Perfect for follow-up

### The Mental Model

```
Think of three buckets:
┌─────┐  ┌─────┐  ┌─────┐
│  0s │  │  1s │  │  2s │
│ Red │  │White│  │Blue │
└─────┘  └─────┘  └─────┘

As you scan, place each element in correct bucket.
But do it IN-PLACE without extra buckets!

That's what the three pointers achieve:
low = where next 0 goes
mid = current element being placed
high = where next 2 goes
```

### When to Use This Pattern

Use three-pointer partitioning when:
- ✅ Need to sort/partition into 3 categories
- ✅ In-place requirement
- ✅ Want one-pass solution
- ✅ Constant space needed
- ✅ Elements have limited distinct values

This elegant algorithm showcases the power of invariant-based thinking and is a classic example of in-place array manipulation!
