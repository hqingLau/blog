## 第一关：Hello World C

输入一行字符串（可能含空格），输出hello world\n，字符串

**Sample Input 0**

```shell
Welcome to C programming.
```

**Sample Output 0**

```shell
Hello, World!
Welcome to C programming.
```

**解决方案**：

```c
int main() 
{
    char s[100];
    // *符: 用以表示该输入项读入后不赋予相应的变量，即跳过该输入值。
    // %[] 扫描字符集合 
    scanf("%[^\n]%*c", &s);  
    printf("%s\n%s\n","Hello, World!",s);
    return 0;
}
```

**知识点**：

空格问题：

```c
char s[20];
scanf("%s",s);
printf("%s\n",s);
// 输入：i love you
// 输出：
// i
```

解决空格问题：

```c
char s[20];
// 读取非换行字符串，%*c表示读取换行，但不赋值
scanf("%[^\n]%*c",s);
printf("%s\n",s);
// 测试如下：
// i love you
// i love you
```

## 第二关：读取字符，字符串，句子

**Sample Input 0**

```shell
C
Language
Welcome To C!!
```

**Sample Output 0**

```shell
C
Language
Welcome To C!!
```

**解决方案**：

```c
char ch;
char str[100];
char sentence[100];

scanf("%c",&ch);
printf("%c\n",ch);

// 这里加个%*c，把结尾的换行消耗了，因为下一个scanf读到换行结束，而
// scanf("%s",s);
// printf("%s\n",s);
// 结果：
//           ccc
// ccc
// 是会跨过前面的空白符的

scanf("%s%*c",str);
printf("%s\n",str);

scanf("%[^\n]",sentence);
printf("%s\n",sentence);
```

## 第三关：两数加减

前两个int，后两个float，输出和and差。

**Sample Input**

```shell
10 4
4.0 2.0
```

**Sample Output**

```shell
14 6
6.0 2.0
```

**解决方案**：

```c
int a,b;
float c,d;
scanf("%d%d",&a,&b);
printf("%d %d\n",a+b,a-b);
scanf("%f%f",&c,&d);
printf("%.1f %.1f\n",c+d,c-d);
```

**知识点**：

还是在`stdin`和`stdout`的输入输出上面。

## 第四关：函数

读取输入四个数字，一个数字一行，输出最大的数字。int类型。

**Sample Input**

```shell
3
4
6
5
```

**Sample Output**

```shell
6
```

**解决方案**：

```c
int max_of_four(int a,int b,int c,int d)
{
    int maxNum = a>b?a:b;
    maxNum = c>maxNum?c:maxNum;
    maxNum = d>maxNum?d:maxNum;
    return maxNum;
}

int main() {
    int a, b, c, d;
    scanf("%d %d %d %d", &a, &b, &c, &d);
    int ans = max_of_four(a, b, c, d);
    printf("%d", ans);
    return 0;
}
```

## 第五关：指针

输入两个数a和b，将更改为a+b, | a-b |。

`void update(int *a,int *b)`

**Sample Input**

```shell
4
5
```

**Sample Output**

```shell
9
1
```

**解决方案：**

```c
#include <stdio.h>

void update(int *a,int *b) {
    int sum = *a + *b;
    int dif = *a - *b;
    dif = dif>0?dif:-dif;
    *a = sum;
    *b = dif; 
}

int main() {
    int a, b;
    int *pa = &a, *pb = &b;
    
    scanf("%d %d", &a, &b);
    update(pa, pb);
    printf("%d\n%d", a, b);

    return 0;
}
```

## 第六关：条件判断

Given a positive integer denoting , do the following:

- If  1-9, print the lowercase English word corresponding to the number (e.g., `one` for , `two` for , etc.).
- If > 9, print `Greater than 9`.

**Sample Input #01**

```shell
8
```

**Sample Output #01**

```shell
eight
```

**Sample Input #02**

```shell
44
```

**Sample Output #02**

```shell
Greater than 9
```

**解决方案：**

```c
char* readline();

int main()
{
    char* n_endptr;
    // 这里的readline是系统定义的，可以自动分配内存
    // 在这里实际并不需要。
    // 也可直接写为：
    // char* n_endptr = malloc(1024);
    // char* n_str = fgets(n_endptr,1023,stdin);
    char* n_str = readline();
    int n = strtol(n_str, &n_endptr, 10);
    // 没读到整数字符串，或者不止有整数字符，退出
    if (n_endptr == n_str || *n_endptr != '\0') { exit(EXIT_FAILURE); }

    char *numStr[10]={"zero","one","two","three","four","five","six",
                        "seven","eight","nine"};
    if(n<9) printf("%s\n",numStr[n]);
    else printf("Greater than 9\n");
    return 0;
}

char* readline() {
    size_t alloc_length = 1024;
    size_t data_length = 0;
    char* data = malloc(alloc_length);

    while (true) {
        char* cursor = data + data_length;
        char* line = fgets(cursor, alloc_length - data_length, stdin);

        if (!line) { break; }

        data_length += strlen(cursor);

        if (data_length < alloc_length - 1 || data[data_length - 1] == '\n') { break; }

        size_t new_length = alloc_length << 1;
        data = realloc(data, new_length);

        if (!data) { break; }

        alloc_length = new_length;
    }

    if (data[data_length - 1] == '\n') {
        data[data_length - 1] = '\0';
    }

    data = realloc(data, data_length);

    return data;
}
```

**知识点：**

strtol：字符串转long

```c
char str[30] = "Hoi 2030300 This is test"; // 0, ptr:Hoi ...
//char str[30] = "2030300 This is test"; // 2030300, ptr: This is test
char *ptr; // 整数结束的字符串
long ret; // 长整数

// 10表示十进制。如二进制
// char str[30] = "10010 This is test";
// ret = strtol(str, &ptr,2); // 18
ret = strtol(str, &ptr, 10);
printf("数字（无符号长整数）是 %ld\n", ret);
printf("字符串部分是 |%s|\n", ptr);
```

## 第七关：for循环

给两个整数，输出区间内的数字英文。1-9，输出对应英文，> 9，输出even还是odd。

**Sample Input**

```shell
8
11
```

**Sample Output**

```shell
eight
nine
even
odd
```

**解决方案：**

```c
int main() 
{
    char *numStr[10]={"zero","one","two","three","four","five","six",
                        "seven","eight","nine"};
    int a, b;
    scanf("%d\n%d", &a, &b);
  	for(int i=a;i<=b;i++) 
    {
        if(i<=9)
        {
            printf("%s\n",numStr[i]);
            continue;
        }
        if(i%2==0) printf("even\n");
        else printf("odd\n");
    }

    return 0;
}

```

## 第八关：五数之和

输入数字：10000-99999

**Sample Input 0**

```shell
10564
```

**Sample Output 0**

```shell
16
```

**解决方案：**

```c
int main() {
    int n;
    int sum = 0;
    int tmp;
    scanf("%d", &n);
    while(n!=0)
    {
        tmp = n%10;
        n /= 10;
        sum+=tmp;
    }
    printf("%d\n",sum);
    return 0;
}
```

## 第九关：位运算

给两个整数，n：一个整数，k：阈值（小于n）。输出i=1 ~ n和i+1 ~ n的and, or, xor的小于k的最大值。

**Sample Input 0**

```shell
5 4
```

**Sample Output 0**

```shell
2
3
3
```

**解决方案：**

```c
void calculate_the_maximum(int n, int k) {
    int maxAnd = 0;
    int maxOr = 0;
    int maxEx = 0;
    int tmp;
    for(int i=1;i<n;i++)
    {
        for(int j=i+1;j<=n;j++)
        {
            tmp = i & j;
            tmp = tmp < k ? tmp:0;
            maxAnd = maxAnd < tmp ? tmp : maxAnd; 
            
            tmp = i | j;
            tmp = tmp < k ? tmp:0;
            maxOr = maxOr < tmp ? tmp : maxOr; 
            
            tmp = i ^ j;
            tmp = tmp < k ? tmp:0;
            maxEx = maxEx < tmp ? tmp : maxEx; 
        }
    }
    printf("%d\n%d\n%d\n",maxAnd,maxOr,maxEx);
}

int main() {
    int n, k;
  
    scanf("%d %d", &n, &k);
    calculate_the_maximum(n, k);
 
    return 0;
}
```

## 第十关：打印模式字符串

**Sample Input 1**

```shell
5
```

**Sample Output 1**

```shell
5 5 5 5 5 5 5 5 5 
5 4 4 4 4 4 4 4 5 
5 4 3 3 3 3 3 4 5 
5 4 3 2 2 2 3 4 5 
5 4 3 2 1 2 3 4 5 
5 4 3 2 2 2 3 4 5 
5 4 3 3 3 3 3 4 5 
5 4 4 4 4 4 4 4 5 
5 5 5 5 5 5 5 5 5
```

**解决方案：**

```c
int min(int a,int b)
{
    return a>b?b:a;
}

int minDistance(int a,int b,int n)
{
    return min(min(min(a,b),2*n-2-a),2*n-2-b);
}

int main() 
{
    int n;
    scanf("%d", &n);
  	// 2 * n - 1 lines
    for(int i=0;i<2*n-1;i++)
    {
        for(int j=0;j<2*n-2;j++)
        {
            printf("%d ",n - minDistance(i,j,n));
        }
        printf("%d\n",n);
    }
    return 0;
}
```

知识点：

> 这一题关键是要找出距离外围的最小距离。

## 第十一关：运盒子

输入盒子个数n，接着n行：长、宽、高

通道高度只有41。输出能过通道的盒子体积（41不能过）。

**Sample Input 0**

```shell
4
5 5 5
1 2 40
10 5 41
7 2 42
```

**Sample Output 0**

```shell
125
80
```

**解决方案：**

```c
#include <stdio.h>
#include <stdlib.h>
#define MAX_HEIGHT 41

typedef struct box
{
	int width;
    int height;
    int length;
}box;


int get_volume(box b) {
	return b.width * b.height * b.length;
}

int is_lower_than_max_height(box b) {
	return b.height < MAX_HEIGHT?1:0;
}

int main()
{
	int n;
	scanf("%d", &n);
	box *boxes = malloc(n * sizeof(box));
	for (int i = 0; i < n; i++) {
		scanf("%d%d%d", &boxes[i].length, &boxes[i].width, &boxes[i].height);
	}
	for (int i = 0; i < n; i++) {
		if (is_lower_than_max_height(boxes[i])) {
			printf("%d\n", get_volume(boxes[i]));
		}
	}
	return 0;
}
```

> 知识点：结构体。

## 第十一关：按三角形面积排序

面积公式：

![image-20211002170435806](https://gitee.com/hqinglau/img/raw/master/img/20211002170437.png)

输入三角形个数，和对应的三条边。

**Sample Input 0**

```shell
3
7 24 25
5 12 13
3 4 5
```

**Sample Output 0**

```shell
3 4 5
5 12 13
7 24 25
```

**解决方案：**

```c
#include <stdio.h>
#include <stdlib.h>
#include <math.h>

struct triangle
{
	int a;
	int b;
	int c;
};

typedef struct triangle triangle;

float areaSquare(const triangle *t)
{
    float p = (t->a + t->b + t->c)/2.0;
    return p*(p - t->a)*(p - t->b)*(p - t->c);
}

int mycmp(const void *a, const void *b)
{
    triangle *ta = (triangle *)a;
    triangle *tb = (triangle *)b;
    return areaSquare(ta)>areaSquare(tb);
}

void sort_by_area(triangle* tr, int n) {
	qsort(tr,n,sizeof(triangle),mycmp);
}

int main()
{
	int n;
	scanf("%d", &n);
	triangle *tr = malloc(n * sizeof(triangle));
	for (int i = 0; i < n; i++) {
		scanf("%d%d%d", &tr[i].a, &tr[i].b, &tr[i].c);
	}
	sort_by_area(tr, n);
	for (int i = 0; i < n; i++) {
		printf("%d %d %d\n", tr[i].a, tr[i].b, tr[i].c);
	}
	return 0;
}
```

**知识点：**

```c
void qsort(void *base, size_t nmemb, size_t size,
              int (*compar)(const void *, const void *));
```
结构体快排。

## 第十二关：反转字符串

**Sample Input 0**

```shell
6
16 13 7 2 1 12 
```

**Sample Output 0**

```shell
12 1 2 7 13 16 
```

**解决方案：**

```c
#include <stdio.h>
#include <stdlib.h>

void swap(int *a,int *b)
{
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

int main()
{
    int num, *arr, i;
    scanf("%d", &num);
    arr = (int*) malloc(num * sizeof(int));
    for(i = 0; i < num; i++) {
        scanf("%d", arr + i);
    }

    for(i = 0;i<(num+1)/2;i++)
    {
        swap(&arr[i],&arr[num-1-i]);
    }

    for(i = 0; i < num; i++)
        printf("%d ", *(arr + i));
    return 0;
}
```

## 第十三关：分词

**Sample Input 0**

```shell
This is C 
```

**Sample Output 0**

```shell
This
is
C
```

**解决方案**：

```c
int main() {
    char *s;
    s = malloc(1024 * sizeof(char));
    scanf("%[^\n]", s);
    s = realloc(s, strlen(s) + 1);
    char *token = strtok(s," ");
    while(token!=NULL)
    {
        printf("%s\n",token);
        token = strtok(NULL," ");
    }
    return 0;
}
```

**知识点：**

strtok，该函数返回被分解的一个子字符串，如果没有可检索的字符串，则返回一个空指针。

使用方法：

- 第1次调用时，第1个参数要传入1个C的字符串，作为要分割的字符串

- 后续调用时，第1个参数设置为空指针NULL

- 上一个被分割的子字符串的位置会被函数内部记住，所以后续调用时，第1个参数设置为NULL

## 第十四关：数字频率

字符串中0-9数字频率，如1出现了两次，output的第二个位置就是2

**Sample Input 0**

```shell
a11472o5t6
```

**Sample Output 0**

```shell
0 2 1 0 1 1 1 1 0 0 
```

**解决方案：**

```c
int main() {
    char str[1024];
    char numCount[10] = {0};
    scanf("%s",str);
    for(int i=0;i<strlen(str);i++)
    {
        if(str[i]>='0'&&str[i]<='9')
        {
            numCount[str[i]-'0']++;
        }
    }
    for(int i=0;i<10;i++) 
        printf("%d ",numCount[i]);
    return 0;
}
```

**知识点：**

数组map的思想。

## 第十五关：字符串排序

词典排序、词典逆序、不同字母的个数排序（相同按字典排序）、长度排序（相同按字典排序）。

**Sample Input 0**

```shell
4
wkue
qoi
sbv
fekls
```

**Sample Output 0**

```shell
fekls
qoi
sbv
wkue

wkue
sbv
qoi
fekls

qoi
sbv
wkue
fekls

qoi
sbv
wkue
fekls
```

**解决方案：**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
int lexicographic_sort(const void* aa, const void* bb) {
    char *a = *(char **)aa;
    char *b = *(char **)bb;
    int alen = strlen(a);
    int blen = strlen(b);
    int minlen = alen>blen?blen:alen;
    for(int i=0;i<minlen;i++)
    {
        if(a[i]!=b[i]) return a[i]-b[i];
    }
    if(alen>blen) return 1;
    return -1;
}

int lexicographic_sort_reverse(const void* aa, const void* bb) {
    char *a = *(char **)aa;
    char *b = *(char **)bb;
    int alen = strlen(a);
    int blen = strlen(b);
    int minlen = alen>blen?blen:alen;
    for(int i=0;i<minlen;i++)
    {
        if(a[i]!=b[i]) return b[i]-a[i];
    }
    if(alen>blen) return -1;
    return 1;
}

int distinctChars(const char *a)
{
    int sum = 0;
    int chrCount[256] = {0};
    for(int i=0;i<strlen(a);i++)
    {
        chrCount[a[i]]++;
    }
    for(int i=0;i<256;i++) 
    {
        if(chrCount[i]!=0) sum++;
    }
    return sum;
}

int sort_by_number_of_distinct_characters(const void* aa, const void* bb) {
    char *a = *(char **)aa;
    char *b = *(char **)bb;
    int na = distinctChars(a);
    int nb = distinctChars(b);
    if(na==nb) return lexicographic_sort(aa,bb);
    return na - nb;
    
}

int sort_by_length(const void* aa, const void* bb) {
    char *a = *(char **)aa;
    char *b = *(char **)bb;
    if(strlen(a)==strlen(b)) return lexicographic_sort(aa, bb);
    return strlen(a)-strlen(b);
}

void string_sort(char** arr,const int len,int (*cmp_func)(const void* a, const void* b)){
    qsort(arr,len,sizeof(char *), cmp_func);
}


int main() 
{
    int n;
    scanf("%d", &n);
  
    char** arr;
	arr = (char**)malloc(n * sizeof(char*));
  
    for(int i = 0; i < n; i++){
        *(arr + i) = malloc(1024 * sizeof(char));
        scanf("%s", *(arr + i));
        *(arr + i) = realloc(*(arr + i), strlen(*(arr + i)) + 1);
    }
  
    string_sort(arr, n, lexicographic_sort);
    for(int i = 0; i < n; i++)
        printf("%s\n", arr[i]);
    printf("\n");

    string_sort(arr, n, lexicographic_sort_reverse);
    for(int i = 0; i < n; i++)
        printf("%s\n", arr[i]); 
    printf("\n");

    string_sort(arr, n, sort_by_length);
    for(int i = 0; i < n; i++)
        printf("%s\n", arr[i]);    
    printf("\n");

    string_sort(arr, n, sort_by_number_of_distinct_characters);
    for(int i = 0; i < n; i++)
        printf("%s\n", arr[i]); 
    printf("\n");
}
```

