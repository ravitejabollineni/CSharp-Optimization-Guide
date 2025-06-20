# LeetCode Problem: Search Insert Position

## Problem Statement

> Given a sorted array of distinct integers and a target value, return the index if the target is found. If not, return the index where it would be if it were inserted in order.
>
> You must write an algorithm with O(log n) runtime complexity.
>
> **Example 1:**
>
> Input: nums = [1,3,5,6], target = 5  
> Output: 2
>
> **Example 2:**
>
> Input: nums = [1,3,5,6], target = 2  
> Output: 1
>
> **Example 3:**
>
> Input: nums = [1,3,5,6], target = 7  
> Output: 4
>
> **Constraints:**
>
> - 1 <= nums.length <= 10⁴
> - -10⁴ <= nums[i] <= 10⁴
> - nums contains distinct values sorted in ascending order.
> - -10⁴ <= target <= 10⁴

---

## Approach 1: Old/Traditional Method

### Solution

```csharp
public int SearchInsert(int[] nums, int target) {
    int k = 0;
    for (int i = 0; i < nums.Length; i++) {
        if (nums[i] == target) return i;
        else if (nums[i] < target) k++;
        else break;
    }
    return k;
}
```

### Explanation

- This approach iterates through the array linearly.
- For each element, it checks if the element equals the target; if so, returns the index.
- If the current element is less than the target, it increments `k`, which tracks the potential insert position.
- If an element greater than the target is found, the loop breaks, and `k` (the insert position) is returned.
- **Time Complexity:** O(n) in the worst case (if target is at the end or not present).
- **Space Complexity:** O(1).

---

## Approach 2: Modern Method (Using Built-in BinarySearch)

### Solution

```csharp
public class Solution {
    public int SearchInsert(int[] nums, int target) {
       int index = Array.BinarySearch(nums, target);
       return index >= 0 ? index : ~index;
    }
}
```

### Explanation

- This solution uses the built-in `Array.BinarySearch` method which performs binary search.
- If the target is found, `Array.BinarySearch` returns its index.
- If not found, it returns the bitwise complement (~) of the index where the target should be inserted.
- The code returns either the found index or the calculated insert position.
- **Time Complexity:** O(log n), as required.
- **Space Complexity:** O(1).

---

## Comparison & Benefits

| Aspect                  | Old Approach                    | Modern Approach                   |
|-------------------------|---------------------------------|-----------------------------------|
| **Readability**         | Simple to understand, verbose   | Concise, leverages built-in method|
| **Performance**         | O(n)                            | O(log n)                          |
| **Use of Language Features**| Basic loops and conditionals | Utilizes built-in BinarySearch    |
| **Maintainability**     | Easy, but more code to manage   | Easier, minimal code              |

### Key Improvements and Optimizations

- **Performance:**  
  The modern approach uses binary search, reducing time complexity from O(n) to O(log n), making it much faster for large arrays.

- **Simplicity & Conciseness:**  
  Leveraging the built-in method reduces boilerplate code and potential for bugs.

- **Best Practices:**  
  Using standard library functions is a modern best practice, as they are well-tested and optimized.

- **Maintainability:**  
  Less code means fewer points of failure and easier updates in the future.

---

## Conclusion

- **Preferred Approach:** The modern approach is superior due to its efficiency, brevity, and reliability. It should be used whenever language/library support is available.
- **When to Use the Old Approach:** Only use the manual approach if you are required to implement binary search yourself or if using a language/environment without built-in search utilities.

---

## References

- [LeetCode Problem: Search Insert Position](https://leetcode.com/problems/search-insert-position/)
- [C# Array.BinarySearch Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.array.binarysearch)
