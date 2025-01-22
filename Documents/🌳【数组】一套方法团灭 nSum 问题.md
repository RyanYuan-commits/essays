# 两数之和
给定一个整数数组 `nums` 和一个整数目标值 `target`，请你在该数组中找出 **和为目标值** `target`  的那 **两个** 整数，并返回它们的值。
首先将数组进行排序，然后按照前面双指针给出的方法来解决即可：
```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Arrays.sort(nums);
        int len = nums.length;
        int left= 0, right = len - 1;

        while (left < right) {
            int sum = nums[left] + nums[right];
            if (sum < target) left++;
            else if (sum > target) right--;
            else return new int[]{nums[left], nums[right]};
        }
        return null;
    }
}
```
让我们继续魔改题目，把这个题目变得更泛化，更困难一点：
**`nums` 中可能有多对儿元素之和都等于 `target`，请你的算法返回所有和为 `target` 的元素对，其中不能出现重复**。
这类题目就需要注意一下去重的问题了：
![[两数之和去重.png|600]]
比如在上图中的情况，当发现 lo + hi 满足要求之后，就需要通过 while 循环来跳过中间的部分，否则就一定会出现重复
```java
List<List<Integer>> twoSumTarget(int[] nums, int target) {
    // nums 数组必须有序
    Arrays.sort(nums);
    int lo = 0, hi = nums.length - 1;
    List<List<Integer>> res = new ArrayList<>();
    while (lo < hi) {
        int sum = nums[lo] + nums[hi];
        int left = nums[lo], right = nums[hi];
        if (sum < target) {
            while (lo < hi && nums[lo] == left) lo++;
        } else if (sum > target) {
            while (lo < hi && nums[hi] == right) hi--;
        } else {
            res.add(Arrays.asList(left, right));
            while (lo < hi && nums[lo] == left) lo++;
            while (lo < hi && nums[hi] == right) hi--;
        }
    }
    return res;
}
```
# 三数之和
对应力扣第 15 题「[三数之和](https://leetcode.cn/problems/3sum/description/)」：给你一个整数数组 `nums` ，判断是否存在三元组 `[nums[i], nums[j], nums[k]]` 满足 `i != j`、`i != k` 且 `j != k` ，同时还满足 `nums[i] + nums[j] + nums[k] == 0` 。请你返回所有和为 `0` 且不重复的三元组。
现在我们想找和为 `target` 的三个数字，那么对于第一个数字，可能是什么？`nums` 中的每一个元素 `nums[i]` 都有可能！
那么，确定了第一个数字之后，剩下的两个数字可以是什么呢？其实就是和为 `target - nums[i]` 的两个数字呗，那不就是 `twoSum` 函数解决的问题么？
```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        Arrays.sort(nums);
        int val = 0, target = 0;
        List<List<Integer>> res = new ArrayList<>();

        for (int i = 0; i < nums.length; i++) {
	        // 先固定第一个元素，然后从这个元素开始向后寻找
            val = nums[i];
            target = -val;
            List<List<Integer>> twoLists = twoSum(nums, i + 1, target);
            if (!twoLists.isEmpty()) {
                for (List<Integer> twoList : twoLists) {
                    twoList.add(val);
                    res.add(twoList);
                }
                while (i < nums.length && nums[i] == val) i = i + 1;
                i = i - 1;
            }
        }
        return res;
    }

    List<List<Integer>> twoSum(int[] nums, int start, int target) {
        int left = start;
        int right = nums.length - 1;
        List<List<Integer>> res = new ArrayList<>();

        while (left < right && right >= 0) {
            int sum = nums[left] + nums[right];
            if (sum < target) left++;
            else if (sum > target) right--;
            else {
                int leftValue = nums[left];
                int rightValue = nums[right];
                ArrayList<Integer> integers = new ArrayList<>();
                integers.add(leftValue);
                integers.add(rightValue);
                res.add(integers);
                while (left < nums.length && nums[left] == leftValue) left++;
                while (right >= 0 && nums[right] == rightValue) right--;
            }
        }
        return res;
    }
}
```

# 四数之和
`4Sum` 完全就可以用相同的思路：穷举第一个数字，然后调用 `3Sum` 函数计算剩下三个数，最后组合出和为 `target` 的四元组。
```java
public class Solution {
    public List<List<Integer>> fourSum(int[] nums, int target) {
        // 数组需要排序
        Arrays.sort(nums);
        int n = nums.length;
        List<List<Integer>> res = new ArrayList<>();
        // 穷举 fourSum 的第一个数
        for (int i = 0; i < n; i++) {
            // 对 target - nums[i] 计算 threeSum
            List<List<Integer>> triples = threeSumTarget(nums, i + 1, target - nums[i]);
            // 如果存在满足条件的三元组，再加上 nums[i] 就是结果四元组
            for (List<Integer> triple : triples) {
                triple.add(nums[i]);
                res.add(triple);
            }
            // fourSum 的第一个数不能重复
            while (i < n - 1 && nums[i] == nums[i + 1]) i++;
        }
        return res;
    }

    // 从 nums[start] 开始，计算有序数组 nums 中所有和为 target 的三元组
    List<List<Integer>> threeSumTarget(int[] nums, int start, long target) {
        int n = nums.length;
        List<List<Integer>> res = new ArrayList<>();
        // i 从 start 开始穷举，其他都不变
        for (int i = start; i < n; i++) {
            ...
        }
        return res;
    }
}
```
# 百数之和
观察上面的解法，其实就是一个典型的递归模型：
比如说我们有五数之和，就固定一个元素，去寻找四数之和，四数之和又需要三数之和。。。
而这个递归的终点就是两数之和来返回结果，比如四数之和问题：
```java
class Solution {
    public List<List<Integer>> fourSum(int[] nums, int target) {
        Arrays.sort(nums);
        return nSum(nums, 4, 0, target);
    }
    List<List<Integer>> nSum(int[] nums, int n, int start, long target) {
        int len = nums.length;
        List<List<Integer>> res = new ArrayList<>();
        if (n < 2 || len < n) return res;
        if (n == 2) {
            return twoSum(nums, start, target);
        } else {
            for (int i = start; i < len; i++) {
                int val = nums[i];
                List<List<Integer>> nSumRes = nSum(nums, n - 1, i + 1, target - val);
                for (List<Integer> list : nSumRes) {
                    list.add(val);
                    res.add(list);
                }
                while (i < len && nums[i] == val) i++;
                i = i - 1;
            }
        }
        return res;
    }
    List<List<Integer>> twoSum(int[] nums, int start, long target) {
        int len = nums.length;
        int left = start;
        int right = len - 1;
        List<List<Integer>> res = new ArrayList<>();

        while (left < right) {
            int sum = nums[left] + nums[right];
            if (sum < target) left++;
            else if (sum > target) right--;
            else {
                int leftVal = nums[left];
                int rightVal = nums[right];
                List<Integer> list = new ArrayList<>();
                list.add(leftVal);
                list.add(rightVal);
                res.add(list);
                while (left < right && nums[left] == leftVal) left++;
                while (left < right && nums[right] == rightVal) right--;
            }
        }

        return res;
    }
}
```