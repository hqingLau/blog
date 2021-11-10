前四十关见：

[Java练习Hackerrank二十道](https://orzlinux.cn/blog/hackerrank_java1.html)

[Java练习Hackerrank另外二十道](https://orzlinux.cn/blog/hackerrank_java2.html)

## 第四十一关：Java Strings Introduction

Java的`BigDecimal`类（[ˈdesɪm(ə)l]）可以处理任意精度的数字。对其操作要用给定的方法，而不是运算符。

逆序排序。

**Sample Input**

```shell
9  // 9个数
-100
50
0
56.6
90
0.12
.12
02.34
000.000
```

**Sample Output**

```shell
90
56.6
50
02.34
0.12
.12
0
000.000
-100
```

**解决方案：**

```java
Arrays.sort(s, 0,n,Collections.reverseOrder((a1, a2) -> {
    BigDecimal a = new BigDecimal(a1);
    BigDecimal b = new BigDecimal(a2);
    return a.compareTo(b);
}));
```

## 第四十二关：Java Primality Test

大数（最多100位）判断质数

**Sample Input**

```shell
13
```

**Sample Output**

```shell
prime
```

**解决方案：**

```java
public class Solution {
    private static BigDecimal one = new BigDecimal(1);
    
    private static boolean isPrime(String n) {
        BigInteger integer = new BigInteger(n);
        return integer.isProbablePrime(1);
    }
    public static void main(String[] args) throws IOException {
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(System.in));
        String n = bufferedReader.readLine();
        bufferedReader.close();
        System.out.println(isPrime(n)?"prime":"not prime");
    }
}
```

**解释：**

```java
// certainty - 这一措施的调用者能容忍的不确定性：如果调用返回true，
// 则此BigInteger是素数超过概率(1 - 1/2确定性)。此方法的执行时间正比于该参数的值。
public boolean isProbablePrime(int certainty) {
    if (certainty <= 0)
        return true;
    BigInteger w = this.abs();
    if (w.equals(TWO))
        return true;
    if (!w.testBit(0) || w.equals(ONE))
        return false;

    return w.primeToCertainty(certainty, null);
}
```

然后100位以内是`Miller-Rabin`素性测试，100位以上是`Miller-Rabin`测试和`Lucas`测试双重判断。

`Miller-Rabin`算法可以在`O(k·log2(n))`的时间内检测一个超级大的正整数n是否是素数，k为自己设定的检测的次数，裸的`Miller-Rabin`算法在验证一个数是否为素数时出错的可能性随着测试次数的增加而降低(不可能降为0)。

## 第四十三关：Java Subarray

打印出子串和为负数的个数，原串数字个数100以内，如：`1 -2 4 - 5 1`，输出为9

![image-20211018121719681](https://gitee.com/hqinglau/img/raw/master/img/20211018121719.png)

**解决方案：**

```java
public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    int n = scanner.nextInt();

    int[] subSum = new int[n];
    subSum[0] = scanner.nextInt();
    for(int i=1;i<n;i++) {
        subSum[i] = subSum[i-1]+scanner.nextInt();
    }
    int result = 0;
    for(int i=0;i<n;i++) {
        if(subSum[i]<0) {
            result++;
        }
        for(int j=i+1;j<n;j++) {
            if(subSum[j]-subSum[i]<0) {
                result++;
            }
        }
    }
    System.out.println(result);
}
```

## 第四十四关：Java Arraylist

二维数组的查询。

**Sample Input**

```shell
5 //5个数组
5 41 77 74 22 44
1 12
4 37 34 36 52
0
3 20 22 33
5 // 5个查询
1 3 // 1,3位置，即74
3 4
3 1 
4 3 // 没有，输出ERROR!
5 5
```

**Sample Output**

```shell
74
52
37
ERROR!
ERROR!
```

**解决方案：**

```java
public static void main(String[] args) {
    Scanner sc = new Scanner(System.in);
    int n = sc.nextInt();
    int[][] arr = new int[n][];
    for(int i=0;i<n;i++) {
        int len = sc.nextInt();
        arr[i] = new int[len];
        for(int j=0;j<len;j++) {
            arr[i][j] = sc.nextInt();
        }
    }
    int qn = sc.nextInt();
    for(int i=0;i<qn;i++) {
        int row = sc.nextInt();
        int col = sc.nextInt();
        if(row>arr.length || col>arr[row-1].length) {
            System.out.println("ERROR!");
        } else {
            System.out.println(arr[row-1][col-1]);
        }
    }
}
```

## 第四十五关：Java BigInteger

大数相加，相乘

**Sample Input**

```shell
1234
20
```

**Sample Output**

```shell
1254  // 和
24680 // 积
```

**解决方案：**

```java
import java.io.*;
import java.util.*;
import java.math.BigInteger;

public class Solution {
    public static void main(String[] args) throws IOException{
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(System.in));
        String a = bufferedReader.readLine();
        String b = bufferedReader.readLine();
        bufferedReader.close();
        
        BigInteger integerA = new BigInteger(a);
        BigInteger integerB = new BigInteger(b);
        System.out.println((integerA.add(integerB)).toString());
        System.out.println((integerA.multiply(integerB)).toString());
    }
}
```

## 第四十六关：Java BitSet

位运算，如第一行`5 4`表明两个5位的数字，有四个操作。

A: 0 0 0 0 0

B: 0 0 0 0 0

**Sample Input**

```shell
5 4
AND 1 2 //and or xor，表示赋值给第一个数字，如and后结果给A，都是0，结果0 0
SET 1 4 // set <> <index>， A: 0 0 0 0 1， 结果 1 0
FLIP 2 2 // 翻转 <> <index>， B: 0 0 1 0 0 ，结果 1 1
OR 2 1 // B=B or A， B: 0 0 1 0 1 结果 1 2
```

**Sample Output**

```shell
0 0
1 0
1 1
1 2
```

**解决方案：**

法一：用boolean数组（一些测试用例会越界，没有做边界检查）

```java
public class Solution {
    private static String operate(boolean[][] arr,String op,int a,int b) {
        switch (op) {
            case "AND":
                for(int i=0;i<arr[0].length;i++) {
                    arr[a-1][i] = arr[a-1][i] & arr[b-1][i];
                }
                break;
            case "OR":
                for(int i=0;i<arr[0].length;i++) {
                    arr[a-1][i] = arr[a-1][i] | arr[b-1][i];
                }
                break;
            case "XOR":
                for(int i=0;i<arr[0].length;i++) {
                    arr[a-1][i] = arr[a-1][i]^ arr[b-1][i];
                }
                break;
            case "FLIP":
                arr[a-1][b] = !arr[a-1][b];
                break;
            case "SET":
                arr[a-1][b] = true;
                break;
            default:
                break;
        }
        int ret1 = 0;
        int ret2 = 0;
        for(int i=0;i<arr[0].length;i++) {
            if(arr[0][i]) {
                ret1++;
            }
            if(arr[1][i]) {
                ret2++;
            }
        }
        return String.format("%d %d",ret1,ret2);

    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int size = sc.nextInt();
        int opNum = sc.nextInt();
        boolean[][] arr = new boolean[2][size];
        for(int i=0;i<opNum;i++) {
            System.out.println(operate(arr,sc.next(),sc.nextInt(),sc.nextInt()));
        }
    }
}
```

法二：BitSet（自带检查）

```java
import java.io.*;
import java.util.*;

public class Solution {
    private static String operate(BitSet[] arr,String op,int a,int b) {
        switch (op) {
            case "AND":
                arr[a-1].and(arr[b-1]);
                break;
            case "OR":
                arr[a-1].or(arr[b-1]);
                break;
            case "XOR":
                arr[a-1].xor(arr[b-1]);
                break;
            case "FLIP":
                arr[a-1].flip(b);
                break;
            case "SET":
                arr[a-1].set(b);
        }
        //返回true的个数
        return String.format("%d %d",arr[0].cardinality(),arr[1].cardinality());

    }

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int size = sc.nextInt();
        int opNum = sc.nextInt();
        BitSet[] arr = new BitSet[]{new BitSet(size),new BitSet(size)};
        for(int i=0;i<opNum;i++) {
            System.out.println(operate(arr,sc.next(),sc.nextInt(),sc.nextInt()));
        }
    }
}
```

法三：函数式接口，源自[tomkri76](https://www.hackerrank.com/challenges/java-bitset/forum/comments/151392)。

Consumer：

```java
StringBuilder stringBuilder = new StringBuilder("orzlinux.cn");

Consumer<StringBuilder> consumer = (str) -> str.append("-hqinglau");
consumer.accept(stringBuilder);
System.out.println(stringBuilder);
// 输出：orzlinux.cn-hqinglau
```

**BiConsumer**: 接收两个参数

```java
StringBuilder stringBuilder = new StringBuilder();

BiConsumer<String,String> consumer = (str1,str2) -> {
    stringBuilder.append("my name is ");
    stringBuilder.append(str1);
    stringBuilder.append(", my web site is ");
    stringBuilder.append(str2);

};
consumer.accept("hqinglau","orzlinux.cn");
System.out.println(stringBuilder);
// 输出：my name is hqinglau, my web site is orzlinux.cn
```

在本题中，可以采用BiConsumer接收两个参数。

```java
Scanner sc = new Scanner(System.in);
int size = sc.nextInt();
int opNum = sc.nextInt();
BitSet[] arr = new BitSet[]{new BitSet(size),new BitSet(size)};
Map<String,BiConsumer<Integer,Integer>> map = new HashMap<>();
map.put("AND",(a,b)->{arr[a-1].and(arr[b-1]);});
map.put("OR",(a,b)->{ arr[a-1].or(arr[b-1]); });
map.put("XOR",(a,b)->{ arr[a-1].xor(arr[b-1]); });
map.put("FLIP",(a,b)->{ arr[a-1].flip(b); });
map.put("SET",(a,b)->{ arr[a-1].set(b); });
for(int i=0;i<opNum;i++) {
    map.get(sc.next()).accept(sc.nextInt(),sc.nextInt());
    System.out.println(String.format("%d %d",arr[0].cardinality(),arr[1].cardinality()));
}
```

## 第四十七关：Java Priority Queue

优先队列

输入格式：

- `ENTER name CGPA id` 放入一个
- `SERVED`取出一个

优先队列先按CGPA排，一样了再按name字典顺序排，输出最后剩下的。

**Sample Input 0**

```shell
12
ENTER John 3.75 50
ENTER Mark 3.8 24
ENTER Shafaet 3.7 35
SERVED
SERVED
ENTER Samiha 3.85 36
SERVED
ENTER Ashley 3.9 42
ENTER Maria 3.6 46
ENTER Anik 3.95 49
ENTER Dan 3.95 50
SERVED
```

**Sample Output 0**

```shell
Dan
Ashley
Shafaet
Maria
```

**解决方案：**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;
/*
 * Create the Student and Priorities classes here.
 */
import java.util.PriorityQueue;
import java.util.Comparator;
class Priorities {
    private PriorityQueue<Student> pq = new PriorityQueue<>(new Comparator<Student>() {
        @Override
        public int compare(Student o1, Student o2) {
            if (o1.getCGPA().equals(o2.getCGPA())) {
                return o1.getName().compareTo(o2.getName());
            }
            return o2.getCGPA().compareTo(o1.getCGPA());
        }
    });
    // lambda
    // private PriorityQueue<Student> pq = new PriorityQueue<>((o1, o2) -> {
    //     if (o1.getCGPA().equals(o2.getCGPA())) {
    //         return o1.getName().compareTo(o2.getName());
    //     }
    //     return o2.getCGPA().compareTo(o1.getCGPA());
    // });
    
    public List<Student> getStudents(List<String> events) {
        // parse and add to priority queue
        List<Student> list = new ArrayList<>();
        for(String ev:events) {
            String[] strs = ev.split(" ");
            if(strs[0].equals("SERVED")) {
                pq.poll();
                continue;
            }
            Student s = new Student(Integer.parseInt(strs[3]),strs[1],Double.parseDouble(strs[2]));
            pq.add(s);
        }
        while (!pq.isEmpty()) {
            list.add(pq.poll());
        }
        return list;
    }
}

class Student{
    private Double CGPA;
    private String name;
    private int id;
    public Student(int i,String n,double c) {
        id = i;
        name = n;
        CGPA = c;
    }
    public String getName() {
        return name;
    }
    public Double getCGPA() {
        return CGPA;
    }
}


public class Solution {
    private final static Scanner scan = new Scanner(System.in);
    private final static Priorities priorities = new Priorities();
    
    public static void main(String[] args) {
        int totalEvents = Integer.parseInt(scan.nextLine());    
        List<String> events = new ArrayList<>();
        
        while (totalEvents-- != 0) {
            String event = scan.nextLine();
            events.add(event);
        }
        
        List<Student> students = priorities.getStudents(events);
        
        if (students.isEmpty()) {
            System.out.println("EMPTY");
        } else {
            for (Student st: students) {
                System.out.println(st.getName());
            }
        }
    }
}
```

## 第四十八关：Java 1D Array (Part 2)

1是栅栏，0是平地，求能不能跳过去，如：

```shell
6 5             game[] size n = 6, leap = 5 (second query)
0 0 0 1 1 1     . . .
```

连续1个数最大为3，一跳为5，输出YES。

**Sample Input**

```shell
STDIN           Function
-----           --------
4               q = 4 (number of queries) 代表4个组合，两行一组
5 3             game[] size n = 5, leap = 3 (first query)
0 0 0 0 0       game = [0, 0, 0, 0, 0]
6 5             game[] size n = 6, leap = 5 (second query)
0 0 0 1 1 1     . . .
6 3
0 0 1 1 1 0
3 1
0 1 0
```

**Sample Output**

```shell
YES
YES
NO
NO
```

**解决方案：**

```java
import java.util.*;

public class Solution {
    private static boolean step(int leap,int[] game,int pos) {
        if(pos>=game.length || (pos+leap>game.length && game[pos]==0)) {
            return true;
        }
        if(pos<0 || game[pos]==1) {
            return false;
        }
        game[pos] = 1; // 标记为1,表示此路不通
        return step(leap,game,pos+1) || step(leap,game,pos-1) || step(leap,game,pos+leap);
    }

    public static boolean canWin(int leap, int[] game) {
        return step(leap,game,0);
    }

    public static void main(String[] args) {
        Scanner scan = new Scanner(System.in);
        int q = scan.nextInt();
        while (q-- > 0) {
            int n = scan.nextInt();
            int leap = scan.nextInt();
            
            int[] game = new int[n];
            for (int i = 0; i < n; i++) {
                game[i] = scan.nextInt();
            }

            System.out.println( (canWin(leap, game)) ? "YES" : "NO" );
        }
        scan.close();
    }
}
```

## 第四十九关：Java List

**Sample Input**

```shell
5 // 初始5个数
12 0 1 78 12 // 初始数组中数字
2 // 两个操作
Insert // 插入
5 23 // 5位置插入23
Delete
0
```

**Sample Output**

```shell
0 1 78 12 23
```

**解决方案：**

```java
import java.io.*;
import java.util.*;

public class Solution {

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        int n = sc.nextInt();
        List<Integer> list = new LinkedList<>();
        for(int i=0;i<n;i++) {
            list.add(sc.nextInt());
        }
        int ops = sc.nextInt();
        for(int i=0;i<ops;i++) {
            switch (sc.next()) {
                case "Delete":
                    list.remove(sc.nextInt());
                    break;
                case "Insert":
                    list.add(sc.nextInt(),sc.nextInt());
                    break;
            }
        }
        String pre = "";
        for(int i=0;i<list.size();i++) {
            System.out.printf("%s%d",pre,list.get(i));
            pre = " ";
        }
        System.out.println();
    }
}
```

## 第五十关：Java Map

**Sample Input**

```shell
3 // 三轮赋值
uncle sam
99912222
tom
11122222
harry
12299933
uncle sam //然后是三轮查找
uncle tom
harry
```

**Sample Output**

```shell
uncle sam=99912222
Not found
harry=12299933
```

**解决方案：**

```java
//Complete this code or write your own from scratch
import java.util.*;
import java.io.*;

class Solution{
    public static void main(String []argh)
    {
        Scanner in = new Scanner(System.in);
        int n=in.nextInt();
        Map<String,Integer> map = new HashMap<>();
        in.nextLine();
        for(int i=0;i<n;i++)
        {
            String name=in.nextLine();
            int phone=in.nextInt();
            map.put(name,phone);
            in.nextLine();
        }
        while(in.hasNext())
        {
            String s=in.nextLine();
            if(map.containsKey(s)) {
                System.out.printf("%s=%d\n",s,map.get(s));
            }
            else {
                System.out.println("Not found");
            }
        }
    }
}
```

## 第五十一关：Java Stack

判断括号是否规范

**Sample Input**

```shell
{}()
({()})
{}(
[]
```

**Sample Output**

```shell
true
true
false
true
```

**解决方案：**

```java
public static void main(String []argh)
{
    Scanner sc = new Scanner(System.in);

    while (sc.hasNext()) {
        Stack<Character> stack = new Stack<>();
        String input=sc.next();
        for(char c:input.toCharArray()) {
            if(stack.empty()) {
                stack.push(c);
                continue;
            }
            switch (c) {
                case '{':
                case '[':
                case '(':
                    stack.push(c);
                    break;
                case ')':
                    if(stack.peek()=='(') {
                        stack.pop();
                    }
                    break;
                case '}':
                    if(stack.peek()=='{') {
                        stack.pop();
                    }
                    break;
                case ']':
                    if(stack.peek()=='[') {
                        stack.pop();
                    }
                    break;
            }
        }
        System.out.println(stack.empty()?"true":"false");
    }
}
```

## 第五十二关：Java Hashset

输入名字，打印个数。

```java
import java.io.*;
import java.util.*;
import java.text.*;
import java.math.*;
import java.util.regex.*;

public static void main(String[] args) {
     Scanner s = new Scanner(System.in);
     int t = s.nextInt();
     String [] pair_left = new String[t];
     String [] pair_right = new String[t];

     for (int i = 0; i < t; i++) {
         pair_left[i] = s.next();
         pair_right[i] = s.next();
     }

     s.close();
     Set<String> set = new HashSet<>();
     for(int i =0; i<t;i++) {
         String p = pair_left[i]+"==="+pair_right[i];
         set.add(p);
         System.out.println(set.size());
     }
 }
```

## 第五十三关：Java Generics

泛型，打印整数数组和字符串数组。

```java
class Printer
{
   public <T> void printArray(T[] arr) {
       for(T t:arr) {
           System.out.println(t);
       }
   }
}
```

## 第五十四关：Java Comparator

```java
class Player{
    String name;
    int score;
    
    Player(String name, int score){
        this.name = name;
        this.score = score;
    }
}
```

分数逆序，名字正序。

**解决方案：**

```java
class Checker implements Comparator<Player> {
    @Override
    public int compare(Player o1, Player o2) {
        if(o1.score==o2.score) {
            return o1.name.compareTo(o2.name);
        }
        return o2.score-o1.score;
    }
}
```

## 第五十五关：Java Inheritance I

基础用法

```java
import java.io.*;
import java.util.*;
import java.text.*;
import java.math.*;
import java.util.regex.*;

class Animal{
	void walk()
	{
		System.out.println("I am walking");
	}
}
class Bird extends Animal
{
	void fly()
	{
		System.out.println("I am flying");
	}
    void sing() {
        System.out.println("I am singing");
    }
}
public class Solution{
   public static void main(String args[]){
	  Bird bird = new Bird();
	  bird.walk();
	  bird.fly();
      bird.sing();
	
   }
}
```

## 第五十六关：Java Inheritance II

加法类继承。

**Output**:

```shell
My superclass is: Arithmetic
42 13 20
```

**解决方案：**

```java
import java.io.*;
import java.util.*;
import java.text.*;
import java.math.*;
import java.util.regex.*;

class Arithmetic {
    public int add(int a,int b) {
        return a+b;
    }
}

class Adder extends Arithmetic {

}
public class Solution{
    public static void main(String []args){
        // Create a new Adder object
        Adder a = new Adder();
        System.out.println("My superclass is: " + a.getClass().getSuperclass().getName());	
        System.out.print(a.add(10,32) + " " + a.add(10,3) + " " + a.add(10,10) + "\n");
     }
}
```

## 第五十七关：Java Abstract Class

抽象类不能创建实例。

```java
abstract class Book{
    String title;
    abstract void setTitle(String s); // 抽象方法，需要子类实现
    String getTitle(){
        return title;
    }
}
```

例子：

```java
import java.util.*;
abstract class Book{
	String title;
	abstract void setTitle(String s);
	String getTitle(){
		return title;
	}
	
}

class MyBook extends Book {
    public void setTitle(String s) {
        title = s;
    }
}

public class Main{
	public static void main(String []args){
		//Book new_novel=new Book(); This line prHMain.java:25: error: Book is abstract; cannot be instantiated
		Scanner sc=new Scanner(System.in);
		String title=sc.nextLine();
		MyBook new_novel=new MyBook();
		new_novel.setTitle(title);
		System.out.println("The title is: "+new_novel.getTitle());
      	sc.close();
		
	}
}
```

## 第五十八关：Java Interface

求除数的和。

**Sample Input**

```shell
6
```

**Sample Output**

```shell
I implemented: AdvancedArithmetic
12 // Divisors of 6 are 1,2,3 and 6. 1+2+3+6=12.
```

**解决方案：**

```java
interface AdvancedArithmetic{
  int divisor_sum(int n);
}

class MyCalculator implements AdvancedArithmetic {
    public int divisor_sum(int n) {
        if(n==1) return 1;
        int result = 0;
        for(int i=1;i<Math.pow(n,0.5);i++) {
            if(n%i==0) {
                result+=i;
                result+=n/i;
            }
        }
        return result;
    }
}
```

## 第五十九关：Java Method Overriding

```java
import java.util.*;
class Sports{

    String getName(){
        return "Generic Sports";
    }
  
    void getNumberOfTeamMembers(){
        System.out.println( "Each team has n players in " + getName() );
    }
}

class Soccer extends Sports{
    @Override
    String getName(){
        return "Soccer Class";
    }
    @Override
    void getNumberOfTeamMembers(){
        System.out.println( "Each team has 11 players in " + getName() );
    }
}

public class Solution{
	
    public static void main(String []args){
        Sports c1 = new Sports();
        Soccer c2 = new Soccer();
        System.out.println(c1.getName());
        c1.getNumberOfTeamMembers();
        System.out.println(c2.getName());
        c2.getNumberOfTeamMembers();
	}
}
```

## 第六十关：Java Instanceof keyword

```java
import java.util.*;


class Student{}
class Rockstar{   }
class Hacker{}


public class InstanceOFTutorial{
	
   static String count(ArrayList mylist){
      int a = 0,b = 0,c = 0;
      for(int i = 0; i < mylist.size(); i++){
         Object element=mylist.get(i);
         if(element instanceof Student)
            a++;
         if(element instanceof Rockstar)
            b++;
         if(element instanceof Hacker)
            c++;
      }
      String ret = Integer.toString(a)+" "+ Integer.toString(b)+" "+ Integer.toString(c);
      return ret;
   }

   public static void main(String []args){
      ArrayList mylist = new ArrayList();
      Scanner sc = new Scanner(System.in);
      int t = sc.nextInt();
      for(int i=0; i<t; i++){
         String s=sc.next();
         if(s.equals("Student"))mylist.add(new Student());
         if(s.equals("Rockstar"))mylist.add(new Rockstar());
         if(s.equals("Hacker"))mylist.add(new Hacker());
      }
      System.out.println(count(mylist));
   }
}
```

