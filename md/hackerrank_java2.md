前二十关见：[Java练习Hackerrank二十道](https://orzlinux.cn/blog/hackerrank_java1.html)

## 第二十一关：Java Strings Introduction

**Sample Input 0**

```shell
hello
java
```

**Sample Output 0**

```shell
9  // 字母个数
No // 第一个字符串是否字典序在第二个后面
Hello Java // 拼接，首字母大写
```

**解决方案**：

```java
public static void main(String[] args) {
    Scanner sc=new Scanner(System.in);
    String A=sc.next();
    String B=sc.next();
    sc.close();
    System.out.println(A.length()+B.length());
    System.out.println(A.compareTo(B)>0?"Yes":"No");
    System.out.printf("%s %s\n",A.substring(0,1).toUpperCase()+A.substring(1),
                      B.substring(0,1).toUpperCase()+B.substring(1));
}
```

> `compareTo`按字典顺序此 `String` 对象位于参数字符串之前，则比较结果为一个负整数。如果按字典顺序此 `String` 对象位于参数字符串之后，则比较结果为一个正整数

## 第二十二关：Java Substring

**Sample Input**

```shell
Helloworld
3 7
```

**Sample Output**

```shell
lowo
```

![image-20211013172632537](https://gitee.com/hqinglau/img/raw/master/img/20211013172632.png)

**解决方案**：

```java
System.out.println(S.substring(start,end));
```

其中，`substring`用法：

```java
public String substring(int beginIndex);
public String substring(int beginIndex, int endIndex);
```

## 第二十三关：Java Substring Comparisons

连续字符字典序排序，输出最大和最小值

**Sample Input 0**

```shell
welcometojava
3
```

**Sample Output 0**

```shell
ava
wel
```

**解决方案**：

```java
import java.util.Scanner;

public class Solution {

    public static String getSmallestAndLargest(String s, int k) {
        int n = s.length();
        if(k>=n||k<=0) {
            return s+"\n"+s;
        } 
        String smallest = s.substring(0,k);
        String largest = s.substring(0,k);
        
        String tmp;
        for(int i=1;i<=n-k;i++) {
            tmp = s.substring(i,i+k);
            if(tmp.compareTo(smallest)<0) {
                smallest = tmp;
            }
            if(largest.compareTo(tmp)<0) {
                largest = tmp;
            }

        }
        
        return smallest + "\n" + largest;
    }


    public static void main(String[] args) {
        Scanner scan = new Scanner(System.in);
        String s = scan.next();
        int k = scan.nextInt();
        scan.close();
      
        System.out.println(getSmallestAndLargest(s, k));
    }
}
```

## 第二十四关：Java String Reverse

判断回文串

```java
public static void main(String[] args) {
    Scanner sc=new Scanner(System.in);
    String A=sc.next();
    System.out.println(A.compareTo(new StringBuffer(A).reverse().toString()) == 0 ? "Yes" : "No");
}
```

## 第二十五关：Java Anagrams

判断变形词（两个单词各个字母的个数一致），忽略大小写。

```java
static boolean isAnagram(String a, String b) {
    int lenA = a.length();
    int lenB = b.length();
    if(lenA!=lenB) {
        return false;
    }
    a = a.toUpperCase();
    b = b.toUpperCase();
    int[] countA = new int[26];
    int[] countB = new int[26];
    for(int i = 0;i<lenA;i++) {
        countA[a.charAt(i)-'A']++;
        countB[b.charAt(i)-'A']++;
    }
    for(int i=0;i<26;i++) {
        if(countA[i]!=countB[i]) {
            return false;
        }
    }
    return true;
}
```

## 第二十六关：Java String Tokens

分词。

`s` is composed of *any* of the following: English alphabetic letters, blank spaces, exclamation points (`!`), commas (`,`), question marks (`?`), periods (`.`), underscores (`_`), apostrophes (`'`), and at symbols (`@`).

**Sample Input**

```shell
He is a very very good boy, isn't he?
```

**Sample Output**

```shell
10
He
is
a
very
very
good
boy
isn
t
he
```

**解决方案：**

```java
public static void main(String[] args) {
    Scanner scan = new Scanner(System.in);
    String s = scan.nextLine().trim();
    if(s.length()==0) {
        System.out.println("0");
        scan.close();
        return;
    }
    String[] strs = s.split("[\\W_]+",0);
    System.out.println(strs.length);
    for(String str:strs) {
        System.out.println(str);
    }
    scan.close();
}
```

## 第二十七关：Pattern Syntax Checker

判断正则式是否有效，用`Pattern.compile`判断。

**Sample Input**

```shell
3
([A-Z])(.+)
[AZ[a-z](a-z)
batcatpat(nat
```

**Sample Output**

```shell
Valid
Invalid
Invalid
```

**解决方案：**

```java
public static void main(String[] args){
    Scanner in = new Scanner(System.in);
    int testCases = Integer.parseInt(in.nextLine());
    while(testCases>0){
        String pattern = in.nextLine();
        try {
            Pattern.compile(pattern);
            System.out.println("Valid");
        } catch (Exception e) {
            System.out.println("Invalid");
        }
        testCases--;
    }
}
```

## 第二十八关：Java Regex

判断ip4地址格式

**Sample Input**

```shell
000.12.12.034
121.234.12.12
23.45.12.56
00.12.123.123123.123
122.23
Hello.IP
```

**Sample Output**

```shell
true
true
true
false
false
false
```

**解决方案：**

```java
class Solution{
    public static void main(String[] args){
        Scanner in = new Scanner(System.in);
        while(in.hasNext()){
            String IP = in.next();
            System.out.println(IP.matches(new MyRegex().pattern));
        }
    }
}

class MyRegex {
    public String pattern;
    
    public MyRegex() {
        pattern = "((25[0-5]|2[0-4][0-9]|1[0-9]{2}|0?[0-9]?[0-9])\\.){3}(25[0-5]|2[0-4][0-9]|1[0-9]{2}|0?[0-9]?[0-9])";
    }
}
```

## 第二十九关：Java Regex 2 - Duplicate Words

**Sample Input**

```shell
5
Goodbye bye bye world world world
Sam went went to to to his business
Reya is is the the best player in eye eye game
in inthe
Hello hello Ab aB
```

**Sample Output**

```shell
Goodbye bye world
Sam went to his business
Reya is the best player in eye game
in inthe
Hello Ab
```

**解决方案：**

```java
public static void main(String[] args) {
    String regex = "\\b(\\w+)(?:\\W+\\1\\b)+";
    Pattern p = Pattern.compile(regex, Pattern.CASE_INSENSITIVE);

    Scanner in = new Scanner(System.in);
    int numSentences = Integer.parseInt(in.nextLine());

    while (numSentences-- > 0) {
        String input = in.nextLine();
        Matcher m = p.matcher(input);
        // Check for subsequences of input that match the compiled pattern
        while (m.find()) {
            input = input.replaceAll(m.group(), m.group(1));
        }
        // Prints the modified sentence.
        System.out.println(input);
    }
    in.close();
}
```

以下解释来自[RodneyShag](https://www.hackerrank.com/challenges/duplicate-word/forum):

```shell
\w ----> A word character: [a-zA-Z_0-9]
\W ----> A non-word character: [^\w]
\b ----> A word boundary
\1 ----> Matches whatever was matched in the 1st group of parentheses, which in this case is the (\w+)
\+ ----> Match whatever it's placed after 1 or more times

The \b boundaries are needed for special cases such as "Bob and Andy" (we don't want to match "and" twice). Another special case is "My thesis is great" (we don't want to match "is" twice).

?: 匹配group的时候不算这个组
group是针对（）来说的，group（0）就是指的整个串，group（1） 指的是第一个括号里的东西，group（2）指的第二个括号里的东西
```

## 第三十关：Valid Username Regular Expression

- [8,30]个字符
- 只能包含字母数字下划线
- 首字母只能字母

**解决方案：**

```java
class UsernameValidator {
    /*
     * Write regular expression here.
     */
    // [a-zA-Z0-9_]同\\w
    public static final String regularExpression = "[a-zA-Z][a-zA-Z0-9_]{7,29}";
}


public class Solution {
    private static final Scanner scan = new Scanner(System.in);
    
    public static void main(String[] args) {
        int n = Integer.parseInt(scan.nextLine());
        while (n-- != 0) {
            String userName = scan.nextLine();

            if (userName.matches(UsernameValidator.regularExpression)) {
                System.out.println("Valid");
            } else {
                System.out.println("Invalid");
            }           
        }
    }
}
```

## 第三十一关：Tag Content Extractor

去掉标签

**Sample Input**

```shell
4
<h1>Nayeem loves counseling</h1>
<h1><h1>Sanjay has no watch</h1></h1><par>So wait for a while</par>
<Amee>safat codes like a ninja</amee>
<SA premium>Imtiaz has a secret crush</SA premium>
```

**Sample Output**

```shell
Nayeem loves counseling
Sanjay has no watch
So wait for a while
None
Imtiaz has a secret crush
```

**解决方案：**

```java
public static void main(String[] args) {
    Scanner in = new Scanner(System.in);
    int testCases = Integer.parseInt(in.nextLine());
    while(testCases>0){
        String line = in.nextLine();

        boolean matchFound = false;
        // ^<限制表示中间不能有标签，就是取最内层合理的标签
        Pattern r = Pattern.compile("<(.+)>([^<]+)</\\1>");
        Matcher m = r.matcher(line);

        while (m.find()) {
            System.out.println(m.group(2));
            matchFound = true;
        }
        if ( ! matchFound) {
            System.out.println("None");
        }
        testCases--;
    }
}
```

## 第三十二关：Java Varargs - Simple Addition

变长参数求和:

```java
public void add(int ...args) {
    int n = args.length;
    int sum = args[0];
    System.out.printf("%d",args[0]);
    for(int i=1;i<n;i++) {
        System.out.printf("+%d",args[i]);
        sum += args[i];
    }
    System.out.printf("=%d\n",sum);
}
```

## 第三十三关：Java Reflection - Attributes

反射，打印出类的所有方法。

```java
public static void main(String[] args){
    Class student = Student.class;
    Method[] methods =student.getDeclaredMethods();

    ArrayList<String> methodList = new ArrayList<>();
    for(Method m:methods){
        methodList.add(m.getName());
    }
    Collections.sort(methodList);
    for(String name: methodList){
        System.out.println(name);
    }
}
```

## 第三十四关：Can You Access?

调用私有内部类的私有方法：

```java
public class Solution {
	public static void main(String[] args) throws Exception {
        try{
			BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
			int num = Integer.parseInt(br.readLine().trim());
			Object o;
            o = new Inner().new Private();
            // 法一：
            Method method = o.getClass().getDeclaredMethod("powerof2", int.class);
            method.setAccessible(true);
            System.out.printf("%d is %s\n",num,(String) method.invoke(o,num));
            // 法二：
            System.out.printf("%d is %s\n",num,((Solution.Inner.Private)o).powerof2(num));
		}//end of try
	}//end of main
	static class Inner{
		private class Private{
			private String powerof2(int num){
				return ((num&num-1)==0)?"power of 2":"not a power of 2";
			}
		}
	}//end of Inner
}//end of Solution
```

## 第三十五关：Prime Checker

```java
// 在 JDK 1.5 之后增加了一种静态导入的语法，用于导入指定类的某个静态成员变量、
// 方法或全部的静态成员变量、方法。如果一个类中的方法全部是使用 static 声明的
// 静态方法，则在导入时就可以直接使用 import static 的方式导入。
import static java.lang.System.in;
class Prime {
    private boolean isPrime(int n) {
        if(n<2) {
            return false;
        }
        if(n==2|n==3) {
            return true;
        }
        for(int i=2;i<Math.pow(n,0.5)+0.5;i++) {
            if(n%i==0) {
                return false;
            }
        }
        return true;
    }
    
    public void checkPrime(int ...args) {
        String seperator = "";
        for(int arg:args) {
            if(isPrime(arg)) {
                System.out.printf("%s%d",seperator,arg);
                seperator=" ";
            }
        }
        System.out.println();
    }
}
```

## 第三十六关：判断重载

这个是里面经常出现的验证代码，用来判断用户是否重载了函数。

获取全部方法后将名字插入set，判断重复。

```java
Method[] methods=Prime.class.getDeclaredMethods();
Set<String> set=new HashSet<>();
boolean overload=false;
for(int i=0;i<methods.length;i++)
{
    if(set.contains(methods[i].getName()))
    {
        overload=true;
        break;
    }
    set.add(methods[i].getName());
}
if(overload)
{
    throw new Exception("Overloading not allowed");
}
```

## 第三十七关：Java Factory Pattern

工厂模式：Food是一个接口，Cake和Pizza是它的实现类，根据输入返回具体的实现类。

```java
class FoodFactory {
    public Food getFood(String order) {
        switch(order) {
            case "cake": return new Cake();
            case "pizza": return new Pizza();
            default: return null;
        }
    }
}
```

## 第三十八关：Java Singleton Pattern

单例模式

```java
class Singleton{
    public String str;
    private static Singleton instance;
    private Singleton() {}
    public static Singleton getSingleInstance() {
        if(instance==null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

双检查模式，适用多线程。如下：

```java
class Singleton{
    public String str;
    private static Singleton instance;
    private Singleton() {}
    public static Singleton getSingleInstance() {
        // 有实例的就不用加锁了，直接返回
        if(instance==null) {
            // 1 没实例，要创建，锁类
            synchronized(Singleton.class) {
                // 防止在 1 处新建了实例
                if(instance==null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

> 采用类锁，无所谓哪个类，都会被拦截，**对于静态方法，由于此时对象还未生成，所以只能采用类锁。**



## 第三十九关：Java Visitor Pattern

访问者模式

[设计模式[23]-访问者模式-Visitor Pattern](https://www.jianshu.com/p/cd17bae4e949)

如老师和学生按成绩和论文评分，结构稳定，对数据的操作易变。更改操作只需统一更改visitor实现类。

![image-20211014225123027](https://gitee.com/hqinglau/img/raw/master/img/20211014225123.png)

## 第四十关：Java Lambda Expressions

给一个接口，实现匿名实现类。

```java
interface PerformOperation {
    boolean check(int a);
}

public PerformOperation isOdd() {
    return new PerformOperation() {
        public boolean check(int a) {
            return a%2==1;
        }
    };
}
```

