# Agoda HackerRank Live Coding Round

## Question 1: Most Applied Promotion

### Problem Statement
Find the most applied promotion ID after each "QUERY" operation in a sequence of "APPLY" and "QUERY" commands.

### Algorithm Steps
1. Initialize a map to count applications for each promotion
2. Track the current most applied promotion and its count
3. For each operation:
   - If "APPLY", update the count and check if it becomes the most applied
   - If "QUERY", add the current most applied promotion to the result array
4. Return the result array

### Solution

```javascript
function mostAppliedPromotion(operations) {
    // Map to store the count of each promotion ID
    const promoMap = new Map();
    let maxPromoId = null;
    let maxPromoCount = 0;
    const result = [];

    // Iterate through each operation
    for (let operation of operations) {
        const parts = operation.split(' ');
        if (parts[0] === 'APPLY') {
            const promoId = parseInt(parts[1]);
            const newCount = (promoMap.get(promoId) || 0) + 1;
            promoMap.set(promoId, newCount);

            // Update maxPromoId and maxPromoCount if needed
            if (
                newCount > maxPromoCount ||
                (newCount === maxPromoCount && (maxPromoId === null || promoId < maxPromoId))
            ) {
                maxPromoId = promoId;
                maxPromoCount = newCount;
            }
        } else if (parts[0] === 'QUERY') {
            result.push(maxPromoId);
        }
    }
    return result;
}
```

**Time Complexity**: O(N), where N is the number of operations  
**Space Complexity**: O(P), where P is the number of unique promotions applied

---

## Question 2: Find Smaller Number Gap

### Problem Statement
For each element in an array, find the gap to the first smaller element to its right.

### Algorithm Steps
1. Iterate from right to left
2. Use a stack to keep track of indices of elements in increasing order
3. For each element, pop from the stack until you find a smaller element
4. The gap is the difference in indices if a smaller element is found, otherwise 0

### Solution

```javascript
function findSmallerNumGap(nums) {
    // Initialize the result array with zeros
    const res = new Array(nums.length).fill(0);
    const stack = [];

    // Traverse the array from right to left
    for (let i = nums.length - 1; i >= 0; i--) {
        // Pop elements from the stack that are not smaller than nums[i]
        while (stack.length > 0 && nums[stack[stack.length - 1]] >= nums[i]) {
            stack.pop();
        }
        // If stack is not empty, the top is the next smaller element
        if (stack.length > 0) {
            res[i] = stack[stack.length - 1] - i;
        }
        // Push current index onto the stack
        stack.push(i);
    }
    return res;
}
```

**Time Complexity**: O(N), since each element is pushed and popped at most once  
**Space Complexity**: O(N), for the stack and result array