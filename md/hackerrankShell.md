## 通用语法

### 插入序列

**Output Format**

```shell
1
3
5
.
.
.
.
.
99  
```

解决方案：

```shell
for i in $(seq 1 2 99)
do
  echo $i
done
```

### 获取输入

**Sample Input 0**

```shell
Dan  
```

**Sample Output 0**

```shell
Welcome Dan  
```

解决方案：

```shell
read userName
echo "Welcome ${userName}"
```

### for loop 打印1-50

```shell
for i in {1..50}
do
  echo $i
done
```

### 输入两个数比较大小

```shell
read x
read y
if [[ $x == $y ]]; then
    echo "X is equal to Y"
elif [[ $x > $y ]]; then
    echo "X is greater than Y"
else
    echo "X is less than Y"
fi
```

### YN case

```shell
read c
case $c in
    [yY]):
        echo "YES"
        ;;
    [Nn]):
        echo "NO"
        ;;
esac
```

### 判断三角形类型

```shell
read a
read b
read c
# 空格matters
if [[ $a == $b && $a == $c ]]; then
    echo "EQUILATERAL"
elif [[ $a == $b || $a == $c || $b == $c ]]; then
    echo "ISOSCELES"
else
    echo "SCALENE"
fi
```

### 输入表达式计算

```shell
read exp
# 直接echo 会存在四舍五入问题
printf "%.3f" $(echo "scale=5; $exp"|bc)
```

### 求平均数

**Sample Input**

```shell
4 #个数
1
2
9
8
```

**Sample Output**

```shell
5.000
```

```shell
read n
sum=0
for i in $(seq 1 $n); do
    read tmp
    sum=$((sum+tmp))
done
printf "%.3f" $(echo "scale=5; $sum/$n"|bc)
```

### 输出两个数的加减乘除

```shell
read a
read b
echo $((a+b))
echo $((a-b))
echo $((a*b))
echo $((a/b))
```

## cut

典型用法：

```shell
$ cat /etc/passwd | head -n5
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
$ cat /etc/passwd | head -n5 | cut -d: -f 7
/bin/bash
/sbin/nologin
/sbin/nologin
/sbin/nologin
/sbin/nologin

```

> -b ：输入每行第n个字符（半角，注意如果有中文将乱码）。可以`cut -b 2,4-7`
>
> `cut -b 13-`代表到末尾
>
> -c ：输入每行第n个字符（适用中文）。
>
> -d ：自定义分隔符，默认为制表符。
>
> -f ：与-d一起使用，指定显示哪个区域。
>
> -n ：取消分割多字节字符（例如中文）。仅和-b标志一起使用。

## tr

```shell
# 压缩重复相邻字符
$ echo aaabbbaacccfddd | tr -s 'abcdf'
abacfd
# 替换
$ echo aaabbbaacccfddd | tr 'a-c' 'A-C'
AAABBBAACCCfddd
# 删除
$ echo aaabbbaacccfddd | tr -d 'a'
bbbcccfddd
# c 补集
$ echo "hello 12 world" | tr -d -c '0-9'
$ 
```

## 数组

**用下标读取某个元素的值**

```shell
arr_element2=${arr_name[2]}，即形式：${数组名[下标]}
```

**用#获取数组长度：**

```shell
${#数组名[@]} 或${#数组名[*]} 
arr_len=${#arr_name[*]}或${#arr_name[@]}
```

**用#获取某元素值的长度**

```shell
arr_elem_len=${#arr_name[index]}  #index为数组下标索引
```

**删除数组**

删除数组某个元素：unset arr_name[index]

删除整个数组：unset arr_number

**数组分片访问**

分片访问形式为：`${数组名[@或*]:开始下标:偏移长度}`

**有用的数组扩展**

数组支持”+=“赋值运算符，利用这一点可以通过这种方式往一个已知数组中更方便的添加元素，特别是往空数组中填充元素时非常有用。

### 读入数组

```shell
n=0
while read a; do
    array[n]=$a;
    n=$((n+1))
done
echo ${array[@]}
```

### 分片

```shell
echo ${array[@]:3:5}
```

**Sample Input**

```shell
Namibia  
Nauru  
Nepal
Netherlands
NewZealand
Nicaragua
Niger
Nigeria
NorthKorea
Norway
```

**Sample Output**

```shell
Netherlands NewZealand Nicaragua Niger Nigeria
```

### 筛选

```shell
echo ${array[@]/*[aA]*/}
```

过滤含有a或A的字符。

### 拼接

拼接三次

```shell
array=($(cat))
ret=(${array[@]} ${array[@]} ${array[@]})

echo ${ret[@]}
```

### 替换

```shell
echo ${array[@]/[A-Z]/.}
```

数组中元素A-Z都替换成`.`。

### 孤独的整数

输入N，然后是N个数，选出其中的奇数个数的那个。

```shell
read N
# 获取数组[1 2 2 1 2]
array=($(cat))
# 转化为字符串"1 2 2 1 2"
str=${array[@]}
# 字符串替换"1^2^2^1^2"
# 然后计算表达式，就是剩下的那个
echo $((${str// /^}))
```

> echo ${string/23/bb}   //abc1bb42341  替换一次
>
> echo ${string//23/bb}  //abc1bb4bb41  双斜杠替换所有匹配
>
> echo ${string/#abc/bb} //bb12342341   #以什么开头来匹配