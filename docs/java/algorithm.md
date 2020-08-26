1.Leetcode 491   标准的DFS列举一个集合中的所有组合

给定一个整型数组, 你的任务是找到所有该数组的递增子序列，递增子序列的长度至少是2。

示例:

输入: [4, 6, 7, 7]
输出: [[4, 6], [4, 7], [4, 6, 7], [4, 6, 7, 7], [6, 7], [6, 7, 7], [7,7], [4,7,7]]

**一个dfs的方式枚举所有子序列的模板**

每一个元素都可以选择或者不选择，2^n种可能

```java
List<List<Integer>> ans = new ArrayList<List<Integer>>();
List<Integer> temp = new ArrayList<Integer>();
public void dfs(int cur, int[] nums) {
    if (cur == nums.length) {
        // 判断是否合法，如果合法判断是否重复，将满足条件的加入答案
        if (isValid() && notVisited()) {
            ans.add(new ArrayList<Integer>(temp));
        }
        return;
    }
    // 选择当前元素
    temp.add(nums[cur]);
    dfs(cur + 1, nums);
    temp.remove(temp.size() - 1);
    // 不选择当前元素
    dfs(cur + 1, nums);
}

```

这道题我去重想了很久  一个优雅的解法就是官方的题解，

但是还有一种方法就是通过hashSet, 将每一层要加入的元素进行去重，不过这样递归模板就得修改一下了

```java
class Solution {
    List<List<Integer>> results;
    public List<List<Integer>> findSubsequences(int[] nums) {
        results = new LinkedList<>();
        dfs(nums, 0, new ArrayList<Integer>());
        return results;
    }

    public void dfs(int[] nums, int index, LinkedList<Integer> result) {
        if (result.size() > 1) {
            List<Integer> oneResult = new ArrayList<>();
            oneResult.addAll(result);
            results.add(oneResult);
        }
		//每一层都判断下
        Set<Integer> visited = new HashSet<>();
        
        for (int i=index; i<nums.length; i++) {
            if (visited.contains(nums[i])) {
                continue;
            }
            visited.add(nums[i]);
            if (result.size() == 0 || result.get(result.size() - 1) <= nums[i]) {
                result.add(nums[i]);
                dfs(nums, i+1, result);
                result.remove(result.size() - 1);
            }
        }
    }
}


```

