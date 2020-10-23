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

**差分数组求频率**

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
  
  求base进制的时候，往往会用到快速幂， 快速计算出x ^n % mod的值：
  
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
  
  

##### DP

###### 树形dp

- 834 [Sum of Distances in Tree](https://leetcode-cn.com/problems/sum-of-distances-in-tree/)  

  不会做，只能学

- 337 [House Robber III](https://leetcode-cn.com/problems/house-robber-iii/)

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

-  [1595. 连通两组点的最小成本](https://leetcode-cn.com/problems/minimum-cost-to-connect-two-groups-of-points/)

#### 数据结构

记录几种常用的，可以用于巧妙解题的数据结构

- **并查集**

```java
public class UnionFind {
    int cnt;
    int[] parent;
    int[] size;
    public UnionFind(int n ){
         cnt = n;
         size = new int[n];
         parent = new int[n];
        for (int i = 0; i < cnt; i++){
            parent[i] = i;
            size[i] = 1;
        }
    }

    int find(int x){
        return parent[x] == x ? x : (parent[x] = find(find(parent[x])));
    }
    //把小树挂在大树下面
  	 void union(int x, int y){
         
        int xP = parent[x];
        int yP = parent[y];
         if (xP != yP){
            if (size[xP] > size[yP]){
            	size[xP] += size[yP];
                parent[yP] = parent[xP];
                cnt--;
        	}
        	else{
            	size[yP] += size[xP];
           		parent[xP] = yP;
            	cnt--;
        	}
        }
      
    }
    
    boolean findAndUnion(int x, int y){
        int xParent = find(x);
        int yParent = find(y);
        if (xParent == yParent) return false;
        union(x, y);
        return true;
    }
}
```

并查集的例题：

 [5128. Graph Connectivity With Threshold](https://leetcode-cn.com/problems/graph-connectivity-with-threshold/)


- **FenwickTree**

又称树状数组。用于求前缀和，O（logn）, 做hard题目的时候遇到过几次，看了好几个小时，终于弄明白了

```java
/*
* 树状数组
* 用于求前缀和 O(logn)
*/
public class FenwickTree {
    //bitarr数组上的每一个位置 都管理一个前缀和：
    // 比如位置110(6)管理的是 (110- lowbit(110), 110]这个区间的前缀和
    //也就是 (100,110];
    private int[] bitarr;
    int len;
    //构建FenwickTree有两种方法：
    //我们只需初始化一个全为0的数组，并对原数组中的每一个
    // 位置对应的数字调用一次update(i, delta)操作即可。这是一个O(nlogn)的建立过程。
    //第二种方法就是下面的O(n)方法
    public  FenwickTree(int[] list){
        bitarr = new int[list.length + 1];
        len = bitarr.length;
        for (int i = 0; i < list.length;i++){
            bitarr[i + 1] = list[i];
        }
        //这里就是对于每个位置i，能包括到i这个位置的所有位置都需要加上位置i上的值
        //这里从1开始，加一次就行；
        for (int i = 1; i < len; i++){
            int j = i + lowbit(i);
            if (j < len){
                bitarr[j] += bitarr[i];
            }
        }
    }
    //idx 原数组的index
    //更新本节点和父节点
    public void update(int idx,  int delta){
        idx++;
        while (idx < len){
            bitarr[idx] += delta;
            idx += lowbit(idx);
        }
    }
    //求和
    public int query(int idx){
        idx++;
        int sum = 0;
        while (idx > 0){
            sum += bitarr[idx];
            idx -= lowbit(idx);
        }
        return sum;
    }
    //找到i最右边那一位的1所代表的数
    private int lowbit(int i) {
        return  i & (-i);
    }
}
```

树状数组的例题：

- [1505. Minimum Possible Integer After at Most K Adjacent Swaps On Dig](https://leetcode-cn.com/problems/minimum-possible-integer-after-at-most-k-adjacent-swaps-on-digits/)

  这道题目暴力是可以想到并且做出来的。测试case不强，没有卡n^2.

  

- **线段树** 

也可以用来求前缀和，还可以用来查找， 我用的不多，代码比较好理解：

```java
public class SegmentTreeNode {
    int start, end, count;
    SegmentTreeNode left, right;

    public SegmentTreeNode(int start, int end){
        this.start = start;
        this.end = end;
        this.count = 0;
    }
    //构建线段树，不断递减区间长度
    private SegmentTreeNode build(int start, int end){
        if (start  > end ) return null;
        SegmentTreeNode root = new SegmentTreeNode(start, end);
        if (start == end) return root;
        int mid = start + (end - start) / 2;
        root.left = build(start, mid);
        root.right = build (mid + 1, end);
        return root;
    }
    private void insert(SegmentTreeNode root, int index, int val){
        if (root.start == index && root.end == index){
            root.count += val;
            return;
        }
        int mid = root.start + ((root.end - root.start) >> 1);
       if (index  > mid && index <= root.end)
            insert(root.right, index , val);
       else if (index <= mid && index >= root.start)
            insert(root.left, index, val);
       root.count  = root.left.count + root.right.count;
    }

    private int count(SegmentTreeNode root, int start, int end){
        if (start > end) return 0;
        if (root == null) return 0;
        if (start == root.start && end == root.end)
            return root.count;
        int mid = root.start + (root.end - root.start) / 2;
        int leftCount = 0, rightCount = 0;
        if (start <= mid){
            if (mid < end)
                leftCount = count(root.left, start, mid);
            else
                leftCount = count(root.left, start, end);
        }
        if (end > mid){
            if (start <= mid)
                rightCount = count(root.right, mid + 1, end);
            else
                rightCount = count(root.right, start, end);
        }
        return leftCount + rightCount;
    }
}
```

#### 算法相关

- 雪花算法

  用于生成分布式ID的，  不需要搭建一个分布式ID的服务，不是基于数据库的，代码很简洁，[github上一个人实现的](https://github.com/beyondfengyu/SnowFlake/blob/master/SnowFlake.java)

  ```java
  public class SnowFlake {
      /**
       * 起始的时间戳
       */
      private final static long START_STMP = 1480166465631L;
  
      /**
       * 每一部分占用的位数
       */
      private final static long SEQUENCE_BIT = 12; //序列号占用的位数
      private final static long MACHINE_BIT = 5;   //机器标识占用的位数
      private final static long DATACENTER_BIT = 5;//数据中心占用的位数
  
      /**
       * 每一部分的最大值
       * 2^n - 1
       */
      private final static long MAX_DATACENTER_NUM = -1L ^ (-1L << DATACENTER_BIT);
      private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
      private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);
  
      /**
       * 每一部分向左的位移
       */
      private final static long MACHINE_LEFT = SEQUENCE_BIT;
      private final static long DATACENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
      private final static long TIMESTMP_LEFT = DATACENTER_LEFT + DATACENTER_BIT;
  
      private long datacenterId;  //数据中心
      private long machineId;     //机器标识
      private long sequence = 0L; //序列号
      private long lastStmp = -1L;//上一次时间戳
  
      public SnowFlake(long datacenterId, long machineId) {
          if (datacenterId > MAX_DATACENTER_NUM || datacenterId < 0) {
              throw new IllegalArgumentException("datacenterId can't be greater than MAX_DATACENTER_NUM or less than 0");
          }
          if (machineId > MAX_MACHINE_NUM || machineId < 0) {
              throw new IllegalArgumentException("machineId can't be greater than MAX_MACHINE_NUM or less than 0");
          }
          this.datacenterId = datacenterId;
          this.machineId = machineId;
      }
  
      /**
       * 产生下一个ID
       *
       * @return
       */
          public synchronized long nextId() {
          long currStmp = getNewstmp();
          if (currStmp < lastStmp) {
              throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
          }
  
          if (currStmp == lastStmp) {
              //相同毫秒内，序列号自增
              sequence = (sequence + 1) & MAX_SEQUENCE;
              //同一毫秒的序列数已经达到最大
              if (sequence == 0L) {
                  currStmp = getNextMill();
              }
          } else {
              //不同毫秒内，序列号置为0
              sequence = 0L;
          }
  
          lastStmp = currStmp;
  
          return (currStmp - START_STMP) << TIMESTMP_LEFT //时间戳部分
                  | datacenterId << DATACENTER_LEFT       //数据中心部分
                  | machineId << MACHINE_LEFT             //机器标识部分
                  | sequence;                             //序列号部分
      }
  
      private long getNextMill() {
          long mill = getNewstmp();
          while (mill <= lastStmp) {
              mill = getNewstmp();
          }
          return mill;
      }
  
      private long getNewstmp() {
          return System.currentTimeMillis();
      }
  
      public static void main(String[] args) {
          SnowFlake snowFlake = new SnowFlake(2, 3);
  
          for (int i = 0; i
               < (1 << 12); i++) {
              System.out.println(snowFlake.nextId());
          }
      }
  }
  ```


- 字符串匹配算法之KMP算法

  指针不会回退，O(n) = m + n;

```java
public class KMP {
    // AS  SA  A是前缀/后缀， A必须是非空串
    //ababab  c    对于这个字符串  next[5] = 4  也就是模式串前字符前一个字符的最长公共前后缀
    //ababab  a
    int[] next;
    public int kmp(String s, String p){
        getNext(p, p);
        int i = 0,  j = 0;
        while (i < s.length() && j < p.length()){
            if (j == -1 || s.charAt(i) == p.charAt(j)){
                i++;
                j++;
            }
            else j = next[j];
        }
        if ( j == p.length()) return i - j;
        return -1;
    }
    private  void getNext(String p1, String p2){
        next = new int[p1.length()];
        next[0] = -1;
        int i = 0, j = -1;
        while ( i < p1.length() - 1){
            if (j == -1 || p1.charAt(i) == p2.charAt(j)){
                i++;
                j++;
                next[i] = j;
            }
            else j = next[j];
        }
    }
}
```

