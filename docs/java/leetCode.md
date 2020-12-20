[TOC]



#### LeetCode

- Leetcode 491   标准的DFS列举一个集合中的所有组合

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

- 5505   有一个整数数组 ``nums ``，和一个查询数组 ``requests`` ，其中 ``requests[i] = [starti, endi] ``。第 i 个查询求 ``nums[starti] + nums[starti + 1] + ... + nums[endi - 1] + nums[endi]``的结果 ，``starti`` 和 ``endi`` 数组索引都是 从 0 开始 的。

你可以任意排列`` nums`` 中的数字，请你返回所有查询结果之和的最大值。

由于答案可能会很大，请你将它对`` 1e9 + 7`` 取余 后返回。

##### 差分数组

```java
思路很简单，就是统计各个位置出现的频率，然后让频率最大的数尽可能取到的值大，但是统计频率这一步容易超时，可以用到查分数组，不需要遍历对每个位置进行统计：
     int mod = (int)1e9 + 7;
    public int maxSumRangeQuery(int[] nums, int[][] requests) {
        int n = nums.length;
        int[] delta = new int[n + 1];
        //这里 差分求频率
        for (int[] arr : requests){
            delta[arr[0]]++;
            delta[arr[1] + 1]--;
        }
        int[] count = new int[n];
        count[0] = delta[0];
        //根据差分数组
        for (int i = 1; i < n; i++){
            count[i] = count[i - 1] + delta[i];
        }
        Arrays.sort(count);
        Arrays.sort(nums);
        int res  = 0;
        for (int i = n - 1; i >= 0; i--){
            if (count[i] == 0) break;
            res = (res + count[i] * nums[i]) % mod;
        }
        return res;
    }
```

[1674. 使数组互补的最少操作次数](https://leetcode-cn.com/problems/minimum-moves-to-make-array-complementary/)

- 332  	“一笔画问题” 欧拉图， 挺有意思的。

  ```java
  class Solution {
      Map<String, Queue<String>> map;
      List<String> res = new ArrayList<>();
      public List<String> findItinerary(List<List<String>> tickets) {
         map = new HashMap<>();
         for (List<String> ticket : tickets){
             Queue<String> pq = map.computeIfAbsent(ticket.get(0), k -> new PriorityQueue<>(String :: compareTo));
             pq.offer(ticket.get(1));
         }
         dfs("JFK");
         Collections.reverse(res);
         return res;
      }
      private void dfs(String start){
          while (map.containsKey(start) && map.get(start).size() > 0){
              String next = map.get(start).poll();
              dfs(next);
          }
          res.add(start);
      }
  }

  ```

- [187. Repeated DNA Sequences](https://leetcode-cn.com/problems/repeated-dna-sequences/)

  **Rabin-Karp算法**对字符串进行hash编码。详见这道题的官方题解

  ```java
  一种初始化map的方式  初始化工厂
   static final Map<Character, String> map = Map.of(
          '2', "abc", '3', "def", '4', "ghi", '5', "jkl",
          '6', "mno", '7', "pqrs", '8', "tuv", '9', "wxyz"
      );
  ```

  一样的题目还有[718. Maximum Length of Repeated Subarray](https://leetcode-cn.com/problems/maximum-length-of-repeated-subarray/)     

  ​                         [1044. Longest Duplicate Substring](https://leetcode-cn.com/problems/longest-duplicate-substring/)
  
  求base进制的时候，往往会用到**快速幂**， 快速计算出x ^n % mod的值：
  
  ```java
  public long qPow(long x, long n){
      long ret = 1;
      while (n != 0){
          if ((n & 1) != 0){
              ret = ret * x % mod;
          }
          x = x * x % mod;
          n >>= 1;
      }
     	return ret;
  }
  ```
  
- [381. Insert Delete GetRandom O(1) - Duplicates allowed](https://leetcode-cn.com/problems/insert-delete-getrandom-o1-duplicates-allowed/)

  没想到。列表其实也能O(1)时间复杂度的删除的，找到对应的index（通过map    ），将其和list末尾元素交换再删除末尾元素，就可以实现O(1)的复杂度删除。

##### DP

###### 我想不到的DP

[1621. Number of Sets of K Non-Overlapping Line Segments](https://leetcode-cn.com/problems/number-of-sets-of-k-non-overlapping-line-segments/)

[1621. 大小为 K 的不重叠线段的数目](https://leetcode-cn.com/problems/number-of-sets-of-k-non-overlapping-line-segments/)

[1639. 通过给定词典构造目标字符串的方案数](https://leetcode-cn.com/problems/number-of-ways-to-form-a-target-string-given-a-dictionary/)

[1643. 第 K 条最小指令](https://leetcode-cn.com/problems/kth-smallest-instructions/)

[514. 自由之路](https://leetcode-cn.com/problems/freedom-trail/)

###### 树形dp

- 834 [Sum of Distances in Tree](https://leetcode-cn.com/problems/sum-of-distances-in-tree/)  

  不会做，只能学

- 337 [House Robber III](https://leetcode-cn.com/problems/house-robber-iii/)

###### 几种类似的dp

[5551. 使字符串平衡的最少删除次数](https://leetcode-cn.com/problems/minimum-deletions-to-make-string-balanced/)

[LCP 19. 秋叶收藏集](https://leetcode-cn.com/problems/UlBDOe/)

跳跃相关的dp   往往也可以用记忆化搜索	

[5552. 到家的最少跳跃次数](https://leetcode-cn.com/problems/minimum-jumps-to-reach-home/)

###### 背包问题

- 416 [ Partition Equal Subset Sum](https://leetcode-cn.com/problems/partition-equal-subset-sum/)

  ```java
  Input: nums = [1,5,11,5]
  Output: true
  Explanation: The array can be partitioned as [1, 5, 5] and [11].
  ```

  每个元素可以选或者不选， 我只想到了暴力dfs。 这其实判断是否可以从数组中选出一些数字，使得这些数字的和等于整个数组的元素和的一半。因此这个问题可以转换成**0-1背包问题**

  ```java
  
  class Solution {
      public boolean canPartition(int[] nums) {
          int sum = 0;
          for (int i : nums) sum += i;
          if (sum % 2 == 1) return false;
          int target = sum / 2;
          boolean[] dp = new boolean[ target + 1];
          dp[0] = true;
          for (int i : nums){
              for (int j = target; j >= i; j--){
                  if ( dp[j - i]) dp[j] = true;
              }
          }
          return dp[target];
      }
  }
  ```

###### 状态压缩

- [1617. Count Subtrees With Max Distance Between Cities](https://leetcode-cn.com/problems/count-subtrees-with-max-distance-between-cities/)

  Floyd算法

  ```java
   //floyd 求任意两点最短距离
          for (int k = 0; k < n; k++){
              for (int i = 0; i < n; i++){
                  for (int j = 0; j < n; j++){
                      if (dist[i][k] != INF  && dist[k][j] != INF){
                          dist[i][j] = Math.min(dist[i][j], dist[i][k] + dist[k][j]);
                      }
                  }
              }
          }
  ```

  

- [1595. 连通两组点的最小成本](https://leetcode-cn.com/problems/minimum-cost-to-connect-two-groups-of-points/)

- [1655. 分配重复整数](https://leetcode-cn.com/problems/distribute-repeating-integers/)

  ```java
  //注意几点
  //1.预处理数据 对所有可能状态  求出该状态下需要的订单综合
          for (int i = 0; i < sum.length ; i++){
              for (int j = 0; j < m; j++){
                  //把 i上的所有位 都理解成一个状态 总共m个状态
                  if ( ((i >> j) &  1) == 1){ 
                      sum[i] += quantity[j];
                  }
              }
          }
       
     //  求一个二进制数的子集 的方法  k = ((k - 1) & j)
  	// 从高位一直到最低位（不包括全0 和最刚开始的j） 从大到小
  	// 相当于是枚举了所有情况
                  for (int k = j; k != 0; k = ((k - 1) & j)){
                      int v = j - k;  // k子集  v 另一个不交子集 
                      if (i == 0){
                          if (v == 0 && cnt[i] >= sum[k]) dp[i][j] = true; 
                      }
                      else if (cnt[i] >= sum[k]){
                          dp[i][j] = dp[i - 1][v];
                      }
                      if (dp[i][j]) break;
                  }
              }
  
  ```

[5619. 最小不兼容性](https://leetcode-cn.com/problems/minimum-incompatibility/)

枚举子集的一个方法：

​	给定一个整数x，如何不重复地枚举x二进制表示的子集y？子集的定义：y的二进制中出现的每一个1，x相同位置也出现，即y & x  = y;

```java
//枚举
for (int y = x; y != 0; y = (x & (y - 1)){
	...
        对应的补集s = x ^ y;
        
}
     
    
```



##### 排序

[1030. 距离顺序排列矩阵单元格](https://leetcode-cn.com/problems/matrix-cells-in-distance-order/)

这道题可以用桶排序

[164. 最大间距](https://leetcode-cn.com/problems/maximum-gap/)

​	经典的桶排序，可以当模板使用。

##### 单调队列

[239. Sliding Window Maximum](https://leetcode-cn.com/problems/sliding-window-maximum/)

​	很经典的单调双端队列

[5614. Find the Most Competitive Subsequence](https://leetcode-cn.com/problems/find-the-most-competitive-subsequence/)

[456. 132模式](https://leetcode-cn.com/problems/132-pattern/)

这道题挺难想的，好好领会吧。

[5631. 跳跃游戏 VI](https://leetcode-cn.com/problems/jump-game-vi/)

模板题，利用单调队列维护最大值，降低复杂度。

##### 贪心思想

leetcode上遇到过的贪心：

[55. Jump Game](https://leetcode-cn.com/problems/jump-game/)

[1024. Video Stitching](https://leetcode-cn.com/problems/video-stitching/)

[948. Bag of Tokens](https://leetcode-cn.com/problems/bag-of-tokens/)

[1642. 可以到达的最远建筑](https://leetcode-cn.com/problems/furthest-building-you-can-reach/)

​	这道题二分也可以。当时想半天知道要用贪心去做，没想到怎么设计这种数据结构！！

[659. 分割数组为连续子序列](https://leetcode-cn.com/problems/split-array-into-consecutive-subsequences/)

 完全没想到

[621. 任务调度器](https://leetcode-cn.com/problems/task-scheduler/)

我想的方法复杂了，get到了那个点，其实完全可以不借助优先队列或者多次数组排序，直接贪心一步到位的。

[1675. 数组的最小偏移量](https://leetcode-cn.com/problems/minimize-deviation-in-array/)

[649. Dota2 参议院](https://leetcode-cn.com/problems/dota2-senate/)

[376. 摆动序列](https://leetcode-cn.com/problems/wiggle-subsequence/)

dp, greedy都可

[5611. 石子游戏 VI](https://leetcode-cn.com/problems/stone-game-vi/)

看错题目了，题目看仔细点。不过就算看对了题目， 感觉也不一定能做出来:sweat:

##### 二分查找/bfs/dfs

[1631. Path With Minimum Effort](https://leetcode-cn.com/problems/path-with-minimum-effort/)

这道题可以用二分/并查集/dijkstra算法，可以仔细看看零神的题解，很经典

[778. 水位上升的泳池中游泳](https://leetcode-cn.com/problems/swim-in-rising-water/)

- LeetCode 1102. 得分最高的路径（优先队列BFS/极大极小化 二分查找）
- LeetCode 410. 分割数组的最大值（极小极大化 二分查找）
- LeetCode 774. 最小化去加油站的最大距离（极小极大化 二分查找）
- LeetCode 875. 爱吃香蕉的珂珂（二分查找）
- LeetCode LCP 12. 小张刷题计划（二分查找）
- LeetCode 1011. 在 D 天内送达包裹的能力（二分查找）
- LeetCode 1062. 最长重复子串（二分查找）
- LeetCode 5438. 制作 m 束花所需的最少天数（二分查找）
- LeetCode 5489. 两球之间的磁力（极小极大化 二分查找）

[5563. 销售价值减少的颜色球](https://leetcode-cn.com/problems/sell-diminishing-valued-colored-balls/)

第一反应是最大堆，贪心做，结果超时。没想到用二分。

[222. 完全二叉树的节点个数](https://leetcode-cn.com/problems/count-complete-tree-nodes/)

```java
//这也能二分  首先确定出节点数目的上下限， 然后进行二分
//这里有一个确定节点是否存在于完全二叉树的方法，通过左右节点0 ，1 对节点进行编号， 确定节点是否存在于完全二叉树中：
// level  层数， 从0开始    k 节点的编号, 从1开始
public boolean exists(TreeNode root, int level, int k) {
        int bits = 1 << (level - 1);
        TreeNode node = root;
        while (node != null && bits > 0) {
            if ((bits & k) == 0) {
                node = node.left;
            } else {
                node = node.right;
            }
            bits >>= 1;
        }

```

![fig1](https://assets.leetcode-cn.com/solution-static/222/1.png)

###### bfs

[5552. 到家的最少跳跃次数](https://leetcode-cn.com/problems/minimum-jumps-to-reach-home/)

[542. 01 矩阵](https://leetcode-cn.com/problems/01-matrix/)

​	很典型的bfs   其实相当于一个多源BFS

[1162. 地图分析](https://leetcode-cn.com/problems/as-far-from-land-as-possible/)

​	题解多重方法  也是一个多源bfs，和542一样

###### dfs

- 一道很经典的判断途中是否有环  可以转化为拓扑排序或者dfs

  [802. 找到最终的安全状态](https://leetcode-cn.com/problems/find-eventual-safe-states/)

  ```java
  //经典dfs判断环：
   public List<Integer> eventualSafeNodes(int[][] graph) {
          List<Integer> res = new ArrayList<>();
          int n = graph.length;
          int[] color = new int[n];
          for (int i = 0; i < n; i++){
              if (!dfs(i, color, graph)){
                  res.add(i);
              }
          }
          return res;
      }
  
      private boolean dfs(int i, int[] color, int[][] graph){
          //还在栈上 说明回到了之前遍历到的节点 有环
          if (color[i] == 1) return true;
          if (color[i] == 2) return false;
          color[i] = 1;
          for (int next : graph[i]){
              if (dfs(next, color, graph)){
                  return true;
              }
          }
          color[i] = 2;
          return false;
      }
  //转化为拓扑排序
   public List<Integer> eventualSafeNodes(int[][] G) {
          int N = G.length;
          boolean[] safe = new boolean[N];
  
          List<Set<Integer>> graph = new ArrayList();
          List<Set<Integer>> rgraph = new ArrayList();
          for (int i = 0; i < N; ++i) {
              graph.add(new HashSet());
              //这个反向图，其实是保存了每一个点的入边的集合
              rgraph.add(new HashSet());
          }
  
          Queue<Integer> queue = new LinkedList();
  
          for (int i = 0; i < N; ++i) {
              if (G[i].length == 0)
                  queue.offer(i);
              for (int j: G[i]) {
                  graph.get(i).add(j);
                  rgraph.get(j).add(i);
              }
          }
  
          while (!queue.isEmpty()) {
              int j = queue.poll();
              safe[j] = true;
             //对于这个点的入边集合， 相当于原图的点的出边， 可以直接从原图中将出边 - 1；
              for (int i: rgraph.get(j)) {
                  graph.get(i).remove(j);
                  if (graph.get(i).isEmpty())
                      queue.offer(i);
              }
          }
  
          List<Integer> ans = new ArrayList();
          for (int i = 0; i < N; ++i) if 
              (safe[i])
              ans.add(i);
  
          return ans;
      }
  
  
```
  

##### 并查集

[5632. 检查边长度限制的路径是否存在。](https://leetcode-cn.com/problems/checking-existence-of-edge-length-limited-paths/)

	离线查找 + 并查集  。感觉挺经典的。离线的意思：「离线」的意思是，对于一道题目会给出若干询问，而这些询问是全部提前给出的，也就是说，你不必按照询问的顺序依次对它们进行处理，而是可以按照某种顺序（例如全序、偏序（拓扑序）、树的 DFS 序等）或者把所有询问看成一个整体（例如整体二分、莫队算法等）进行处理。
	与「离线」相对应的是「在线」思维，即所有的询问是依次给出的，在返回第 k 个询问的答案之前，不会获得第 k+1个询问。
	来源：zerotrac



##### 排列组合

[5555. 统计字典序元音字符串的数目](https://leetcode-cn.com/problems/count-sorted-vowel-strings/)

也可以用dp dfs   重点理解z神的排列组合方法

[1643. 第 K 条最小指令](https://leetcode-cn.com/problems/kth-smallest-instructions/)

##### 前缀和相关

[1658. 将 x 减到 0 的最小操作数](https://leetcode-cn.com/problems/minimum-operations-to-reduce-x-to-zero/)

[1423. 可获得的最大点数](https://leetcode-cn.com/problems/maximum-points-you-can-obtain-from-cards/)

##### 分治思想

[395. 至少有K个重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-with-at-least-k-repeating-characters/)

​	多路分治

##### 涉及位运算

[5620. 连接连续二进制数字](https://leetcode-cn.com/problems/concatenation-of-consecutive-binary-numbers/)

一拿到这道题，想了一个小时，只想到用蠢办法一个一个的暴力计算，结果TLE, 没有仔细思考到抽象规律。

其实一直TLE,就应该换一种思路想， 而不是耗费时间在原有想法上优化，怎么优化都优化不出来，必须跳出来，重新看题目，思考题目中这些变换后面的本质。

**快速幂**

```java
public long qPow(long x, long n){
    long ret = 1;
    while (n != 0){
        if ((n & 1) != 0){
            ret = ret * x % mod;
        }
        x = x * x % mod;
        n >>= 1;
    }
   	return ret;
}
```



