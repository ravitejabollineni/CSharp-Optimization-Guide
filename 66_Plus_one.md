# LeetCode Problem: Plus One

## Problem Statement

> You are given a large integer represented as an integer array `digits`, where each `digits[i]` is the ith digit of the integer. The digits are ordered from most significant to least significant in left-to-right order. The large integer does not contain any leading 0's.
>
> Increment the large integer by one and return the resulting array of digits.
>
> **Example 1:**
>
> Input: digits = [1,2,3]  
> Output: [1,2,4]  
> Explanation: The array represents the integer 123. Incrementing by one gives 123 + 1 = 124.
>
> **Example 2:**
>
> Input: digits = [4,3,2,1]  
> Output: [4,3,2,2]  
> Explanation: The array represents the integer 4321. Incrementing by one gives 4321 + 1 = 4322.
>
> **Example 3:**
>
> Input: digits = [9]  
> Output: [1,0]  
> Explanation: The array represents the integer 9. Incrementing by one gives 9 + 1 = 10.
>
> **Constraints:**
>
> - 1 <= digits.length <= 100
> - 0 <= digits[i] <= 9
> - digits does not contain any leading 0's.

---

## Approach 1: Convert to Integer, Add One, Convert Back

### Solution

```csharp
public class Solution {
    public int[] PlusOne(int[] digits) {
        int sum = 0;
        for (int i = 0; i < digits.Length; i++)
        {
            sum += digits[i] * (int)Math.Pow(10, digits.Length-1-i);
        }
        sum += 1;
        return Math.Abs(sum)
            .ToString()
            .Select(c => c - '0')
            .ToArray();
    }
}
```

### Explanation

- This approach first converts the array of digits into an integer (`sum`) by multiplying each digit with its corresponding power of ten.
- It then increments the sum by one.
- The resulting integer is converted back to an array of digits by converting to a string and mapping each character back to an integer.
- **Time Complexity:** O(n), but with expensive operations: integer overflow is possible for large arrays, and string conversion is costly.
- **Limitations:** This method does not handle very large numbers (beyond `int` or even `long` range), which violates the problem's constraints for large input sizes (up to 100 digits).

---

## Approach 2: Optimized Digit-by-Digit Addition (No Integer Conversion)

### Solution

```csharp
public class Solution {
    public int[] PlusOne(int[] digits) {
        for (int i = digits.Length - 1; i >= 0; i--) {
            if (digits[i] < 9) {
                digits[i]++;
                return digits;
            }
            digits[i] = 0;
        }
        // If all digits were 9, we need an extra digit at the front
        int[] result = new int[digits.Length + 1];
        result[0] = 1;
        return result;
    }
}
```

### Explanation

- The algorithm starts from the least significant digit (the end of the array) and moves leftwards.
- If the current digit is less than 9, it increments the digit by one and returns the result.
- If the digit is 9, it sets it to 0 and continues to the next digit to the left.
- If all digits are 9 (e.g., `[9,9,9]`), a new array is created with an extra leading 1 (e.g., `[1,0,0,0]`).
- **Time Complexity:** O(n), where n is the length of the digits array.
- **Space Complexity:** O(n) (worst case, when all digits are 9).

---

## Comparison & Benefits

| Aspect                  | Old Approach                        | Optimized Approach                        |
|-------------------------|-------------------------------------|-------------------------------------------|
| **Performance**         | O(n) but with expensive conversions | O(n) with direct digit manipulation       |
| **Handles Large Input** | ❌ (Integer overflow possible)       | ✅ (Works for any length within constraints)|
| **Readability**         | Simple but not robust               | Clean, concise, robust                    |
| **Best Practices**      | Relies on string/integer conversion | Efficient in-place digit operation        |
| **Maintainability**     | Fragile for big numbers             | Reliable for all valid input              |

### Key Improvements and Optimizations

- **Overflow Safe:** The optimized approach never converts the digits to an integer, so it handles arbitrarily large numbers.
- **Efficiency:** No string or integer conversions are needed, only direct array manipulation.
- **Simplicity:** The logic is straightforward and directly matches the problem's requirements.
- **Space:** Only allocates a new array in the rare case when all digits are 9.

---

## Conclusion

- **Preferred Approach:** The optimized approach is best for this problem, as it is efficient, robust, and easily handles the largest valid inputs.
- **When to Use the Old Approach:** Only for educational purposes or very small input sizes where performance and overflow are not concerns.

---

## References

- [LeetCode Problem: Plus One](https://leetcode.com/problems/plus-one/)
