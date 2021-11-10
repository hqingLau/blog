## **数组中重复的数字**

找出数组中重复的数字。在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。

可以开一个数组，时间空间复杂度都是O(N)。

根据题意条件，本地交换

![image-20210906230735778](https://gitee.com/hqinglau/img/raw/master/img/20210906230735.png)

```java
class Solution {
    public int findRepeatNumber(int[] nums) {
        int i = 0;
        while(i<nums.length) {
            if(nums[i]==i) {
                i++;
                continue;
            }
            int n = i;
            int a;
            int b;
            do {
                a = nums[n];
                b = nums[a];
                if(a==b) return a;
                nums[n] = b;
                nums[a] = a;
                if(n==b) break;
            }while(true);
        }
        return -1;
    }
}
```



## 用两个栈实现队列：

第一个想法是一个栈作为主栈，另一个作为辅助，主站全部弹出到辅助栈，再全部压回去。效率太低。

高效做法是辅助栈到弹出完为止不用变，`pop`就行了，没了再把主栈全部压进去。

```java
class CQueue {
    private Deque<Integer> q1; //主
    private Deque<Integer> q2;
    public CQueue() {
        q1 = new LinkedList<>();
        q2 = new LinkedList<>();
    }
    
    public void appendTail(int value) {
        q1.push(value);
    }

    public int deleteHead() {
        int x = -1;
        if(q2.isEmpty()) {
            while(!q1.isEmpty()) {
                q2.push(q1.pop());
            }
        }
        if(!q2.isEmpty())
            x = q2.pop();
        return x;
    }
}
```

## 斐波那契数列：

递归或者动态规划

```java
class Solution {
    public int fib(int n) {
        if(n==0) return 0;
        if(n==1) return 1;
        int a = 0;
        int b = 1;
        int ret = -1;
        for(int i=2;i<=n;i++)
        {
            ret = (a+b)%1000000007;
            a = b;
            b = ret;
        }
        return ret;
    }
}
```