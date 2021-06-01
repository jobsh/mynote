# leetcode 刷题

## 算法

### 1. 两数之和  

>  给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。你可以假设每种输入只会对应一个答案。但是，数组中同一个元素不能使用两遍。
>
> 链接：<https://leetcode-cn.com/problems/two-sum>

- solution

  - 方法一

    ```java
    public static int[] twoSum1(int[] nums, int target) {
            int[] arr = new int[2];
            for (int i = 0; i < nums.length-1; i++){
                for (int j = i + 1; j < nums.length && i != j; j++) {
                    if (nums[i] + nums[j] == target){
                        arr[0] = i;
                        arr[1] = j;
                        return arr;
                    }
                }
            }
            return arr;
    }		
    ```

  * 方法二

    ```java
    public static int[] twoSum(int[] nums, int target) {
            int[] arr = new int[2];
            Map map = new HashMap();
            for (int i = 0; i < nums.length; i++) {
                int diff = target - nums[i];
                if (map.containsKey(diff)){
                    arr[0] = (Integer) map.get(diff);
                    arr[1] = i;
                    return arr;
                } else {
                    map.put(nums[i], i);
                }
            }
            return arr;
    }
    ```

    

### 2. 斐波那契数列

> 写一个函数，输入 n ，求斐波那契（Fibonacci）数列的第 n 项。斐波那契数列的定义如下：F(0) = 0,   F(1) = 1，F(N) = F(N - 1) + F(N - 2), 其中 N > 1.斐波那契数列由 0 和 1 开始，之后的斐波那契数就是由之前的两数相加而得出。答案需要取模 1e9+7（1000000007），如计算初始结果为：1000000008，请返回 1。来源：力扣（LeetCode）链接：<https://leetcode-cn.com/problems/fei-bo-na-qi-shu-lie-lcof>

- solution

  - 方法一

    ```java
    public int countF(int num){
            if (num == 0) {
                return 0;
            }
    
            int[] fa = new int[num + 1];
            fa[1] = 1;
    
            if (num < 2) {
                return fa[num];
            }
            if (fa[num] == 0) {
                for (int i = 2; i < num + 1; i++) {
                    fa[i] = fa[i -1] + fa[i - 2];
                }
            }
            return fa[num];
    }
    ```

    

  - 方法二

    ```java
    public int countF2(int num){
            if (num < 2) return num;
            int first = 0;
            int second = 1;
            int temp = 0;
            for (int i = 2; i < num + 1; i++){
                temp = (first + second) % 1000000007;
                first = second;
                second = temp;
            }
            return temp;
    }
    ```

    

### 3. 移动零

> 给定一个数组 nums，编写一个函数将所有 0 移动到数组的末尾，同时保持非零元素的相对顺序。示例:输入: [0,1,0,3,12]输出: [1,3,12,0,0]说明:必须在原数组上操作，不能拷贝额外的数组。尽量减少操作次数。来源：力扣（LeetCode）
>
> 链接：<https://leetcode-cn.com/problems/move-zeroes>著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

* solution 
  * solution 1

    ```java
    public void moveZeroes1(int[] nums) {
    
            /** 方法一：采用类似冒泡的放大把非零元素移动到前面
             *  1. 设定标志位： 0 移动到的下一个位置
             *  2. 使用for循环不断地比较，如果不等于零则往前移
             */
            // [1,2,0,0,4,5,0,7,0]
            // [1,2,4,0,0,5,0,7,0]
            // [1,2,4,5,0,0,0,7.0]
            // [1,2,4,5,7,0,0,0,0]
            int flag = 0;
            for (int i = 0; i < nums.length; i++) {
    
                if (nums[i] == 0) {
                    flag = i;
                    for (int j = i + 1; j< nums.length; j++){
    
                        // 找到第一个flag后不为0的数
                        if (nums[j] != 0){
                            // 交换
                            nums[flag] = nums[j];
                            nums[j] = 0;
                            break;
                        }
                    }
    
                }
            }
    
            System.out.println(Arrays.toString(nums));
        }
    ```

  * solution 2

    ```java
    private void moveZeroes3(int[] arr) {
            int j = 0;
            for (int i = 0; i < arr.length; i++) {
                if (arr[i] != 0){
                    if (i != j){
                        arr[j] = arr[i];
                        arr[i] = 0;
                    }
                    j++;
                }
            }
            System.out.println(Arrays.toString(arr));
        }
    ```

    

### 4. container of water

- solution

  - solution 1 : 夹逼

    ```java
    public static int maxArea2(int[] height) {
            int max = 0;
            for (int i = 0, j = height.length - 1; i < j; ) {
                int h = height[i] < height[j] ? height[i++] : height[j--];
    //          int h = Math.min(height[i++], height[j--]);
                // 移动以后变少了，所以要 +1
                max = Math.max(h * (j - i + 1),max);
            }
    
            return max;
    }
    ```

    

  - solution 2 ： 暴力遍历

    ```java
    public int maxArea(int[] height) {
            int h = 0;
            int v = 0;
            int max = 0;
            for (int i = 0; i < height.length - 1; i++) {
                for (int j = i + 1; j < height.length; j++) {
                    h = Math.min(height[i],height[j]);
                    v = j - i * h;
                    if (v > max) {
                        max = v;
                    }
                }
            }
            return max;
    }
    ```

    

### 5. climbing - stairs

> 假设你正在爬楼梯。需要 n 阶你才能到达楼顶。
>
> 每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？
>
> 链接：https://leetcode-cn.com/problems/climbing-stairs

* solution : 数学归纳

  ```java
  class Solution {
      // n 从 1 开始
      public int climbStairs(int n) {
          if (n < 3) {
              return n;
          }
          
          int first = 1;
          int second = 2;
          int num = 0;
          
          for (int i = 2; i < n; i++) {
              num = first + second;
              first = second;
              second = num;
          }
          
          return num;
      }
  }	
  ```

### 6.  三数之和(3sum)

> 给你一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？请你找出所有满足条件且不重复的三元组。
>
> 注意：答案中不可以包含重复的三元组。
> 链接：https://leetcode-cn.com/problems/3sum

* solution 1：三层for循环（输出会有重复，暂时不知道如何解决）

  ```java
  public List<List<Integer>> threeSum(int[] nums) {
          List<List<Integer>> intArrayList = new ArrayList<>();
  
          for (int i = 0; i < nums.length - 2; i++) {
              for (int j = i + 1; j < nums.length - 1; j++) {
                  for (int k = j + 1; k < nums.length; k++) {
                      int[] ints = new int[3];
                      List<Integer> integerList = new ArrayList<>(3);
                      int sum = nums[i] + nums[j] + nums[k];
                      if (sum == 0) {
                          integerList.add(nums[i]);
                          integerList.add(nums[j]);
                          integerList.add(nums[k]);
  
                          intArrayList.add(integerList);
                      }
                  }
              }
  
          }
          return intArrayList;
      }
  ```

* solution 2：双指针法

  ```java
  class Solution {
      public List<List<Integer>> threeSum(int[] nums) {
          Arrays.sort(nums);
          List<List<Integer>> res = new ArrayList<>();
          for(int k = 0; k < nums.length - 2; k++){
              if(nums[k] > 0) break;
              if(k > 0 && nums[k] == nums[k - 1]) continue;
              int i = k + 1, j = nums.length - 1;
              while(i < j){
                  int sum = nums[k] + nums[i] + nums[j];
                  if(sum < 0){
                      while(i < j && nums[i] == nums[++i]);
                  } else if (sum > 0) {
                      while(i < j && nums[j] == nums[--j]);
                  } else {
                      res.add(new ArrayList<Integer>(Arrays.asList(nums[k], nums[i], nums[j])));
                      while(i < j && nums[i] == nums[++i]);
                      while(i < j && nums[j] == nums[--j]);
                  }
              }
          }
          return res;
      }
  }
  ```

### 7. 循环链表

* solution 1：

  ```java
  // 暴力，循环遍历
      public boolean hasCycle(ListNode head) {
          List list = new ArrayList();
          while (head != null) {
              if (list.contains(head))
                  return true;
              list.add(head);
              head = head.next;
          }
          return false;
      }
  ```

* solution 2：快慢指针

  ```java
  public boolean hasCycle2(ListNode head) {
          if (head == null || head.next == null){
              return false;
          }
          ListNode slow = head;
          ListNode fast = head.next;
          while (slow != fast) {
              if (fast == null || fast.next == null)
                  return false;
              slow = slow.next;
              fast = fast.next.next;
          }
          return true;
  }
  ```

  

## 数据库

### 1. 组合两个表  