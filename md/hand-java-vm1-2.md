## 一、命令行工具

安装Java和Go

```shell
D:\>java -version
java version "1.8.0_291"
Java(TM) SE Runtime Environment (build 1.8.0_291-b10)
Java HotSpot(TM) Client VM (build 25.291-b10, mixed mode)

D:\>go version
go version go1.17.2 windows/386
```

设置GOPATH：

```shell
D:\>go env GOPATH
D:\go\workspace
```

Java命令

```shell
java [-options] class [args]
java [-options] -jar jarfile [args]
javaw [-options] class [args]
javaw [-options] -jar jarfile [args]
```

javaw命令不显示命令行窗口，适合GUI应用。

**目录**：

![image-20211012140908828](https://gitee.com/hqinglau/img/raw/master/img/20211012140908.png)

cmd.go

```go
package main

import (
	"flag"
	"fmt"
	"os"
)

type Cmd struct {
	helpFlag    bool // 显示帮助信息 标志位
	versionFlag bool // 显示版本
	cpOption    string // classpath
	class       string // 类名
	args        []string // 给main的参数
}

func parseCmd() *Cmd {
	cmd := &Cmd{}

	flag.Usage = printUsage
	flag.BoolVar(&cmd.helpFlag,"help",false,"print help message")
	flag.BoolVar(&cmd.helpFlag,"?",false,"print help message")
	flag.BoolVar(&cmd.versionFlag,"version",false,"print version and exit")
	flag.StringVar(&cmd.cpOption,"classpath","","classpath")
	flag.StringVar(&cmd.cpOption,"cp","","classpath")
	//定义好命令行 flag 参数后，需要通过调用 flag.Parse() 来对命令行参数进行解析
	flag.Parse()

	args := flag.Args()
	if len(args) > 0 {
		cmd.class = args[0]
		cmd.args = args[1:]
	}

	return cmd
}

func printUsage() {
	fmt.Printf("Usage: ·%s [-options] class [args...]\n",os.Args[0])
}
```

main.go

```go
package main

import (
   "fmt"
)

func main() {
   cmd := parseCmd()
   if cmd.versionFlag {
      fmt.Println("Version 0.0.1")

   } else if cmd.helpFlag || cmd.class == "" {
      printUsage()
   } else {
      startJVM(cmd)
   }
}
/**
运行虚拟机
 */
func startJVM(cmd *Cmd) {
   fmt.Printf("classpath: %s class: %s args:%v\n", cmd.cpOption, cmd.class, cmd.args)
}
```

**测试**：

go install 之后在bin运行

```shell
D:\go\workspace\bin>main.exe -help
Usage: ·main.exe [-options] class [args...]

D:\go\workspace\bin>main.exe -version
Version 0.0.1

D:\go\workspace\bin>main.exe -cp foo/bar myApp arg1 arg2
classpath: foo/bar class: myApp args:[arg1 arg2]
```

## 二、搜索class文件

目录：

![image-20211013125824834](https://gitee.com/hqinglau/img/raw/master/img/20211013125824.png)

如Oracle的虚拟机：

- 启动类路径：jre\lib
- 扩展类路径：jre\lib\ext
- 用户类路径

用户类路径默认"."。可以通过CLASSPATH环境变量设置，也可以通过-classpath/-cp覆盖。可以指定目录、jar、zip，或者多个目录、文件（分隔符分开）。

指定`jre`路径：

![image-20211012143120225](https://gitee.com/hqinglau/img/raw/master/img/20211012143120.png)

### 实现类路径

由启动类路径、扩展类路径、用户类路径组成，三者又可以由更细的路径组成。

当输入一条语句，如：

```shell
D:\go\workspace\bin>main.exe -Xjre "C:\Program Files\Java\jdk1.8.0_181\jre" java.lang.Object
```

运行过程：

```go
/**
运行虚拟机
 */
// lll
func startJVM(cmd *Cmd) {
   cp:= classpath.Parse(cmd.XjreOption,cmd.cpOption)
   fmt.Printf("classpath:%v class:%v args:%v\n",
      cp,cmd.class,cmd.args)
   className := strings.Replace(cmd.class,".","/",-1)
   classData,_,err:=cp.ReadClass(className)
   if err!=nil {
      fmt.Printf("could not find or reload main class %s\n",cmd.class)
      return
   }
   fmt.Printf("class data:%v\n",classData)
}
```

`parse`解析`jre`路径和用户`classpath`路径。`ReadClass`从三个路径寻找类，找到返回。路径抽象为一个接口。

Entry.go:

```go
package classpath

import (
	"os"
	"strings"
)

// 系统路径分隔符
const pathListSeparator = string(os.PathListSeparator)

type Entry interface {
	// 寻找和加载class，如java/lang/Object.class
	readClass(className string) ([]byte, Entry, error)
	// String 返回变量的字符串形式
	String() string
}

// 统一的newEntry类，具体实现内部细分
func newEntry(path string) Entry {
	if strings.Contains(path,pathListSeparator) {
		// 由分割符分隔的多个路径组合
		return newCompositeEntry(path)
	}
	if strings.HasSuffix(path,"*") {
		return newWildcardEntry(path)
	}
	if strings.HasSuffix(path,".jar") || strings.HasSuffix(path,".JAR") ||
		strings.HasSuffix(path,".zip") || strings.HasSuffix(path,".ZIP") {
		return newZipEntry(path)
	}
	return newDirEntry(path)
}
```

一个结构体`Classpath`管理三个`entry`

其中，classpath.go:

```go
package classpath

import (
   "os"
   "path/filepath"
)

type ClassPath struct {
   bootClasspath Entry // 启动类路径：jre\lib
   extClassPath  Entry // 扩展类路径：jre\lib\ext
   userClasspath Entry // 用户类路径
}

func Parse(jreOption, cpOption string) *ClassPath {
   cp := &ClassPath{}
   cp.parseBootAndExtClasspath(jreOption)
   cp.parseUserClasspath(cpOption)
   return cp
}

func (self *ClassPath) ReadClass(className string) ([]byte, Entry, error) {
   className = className+".class"
   if data,entry,err:=self.bootClasspath.readClass(className);err==nil {
      return data,entry,err
   }
   if data,entry,err:=self.extClassPath.readClass(className);err==nil {
      return data,entry,err
   }
   return self.userClasspath.readClass(className)
}

func (self *ClassPath) String() string {
   return self.userClasspath.String()
}

// 新建通配符entry
func (self *ClassPath) parseBootAndExtClasspath(jreOption string) {
   jreDir := getJreDir(jreOption)

   // jre/lib/*
   jreLibPath := filepath.Join(jreDir, "lib", "*")
   self.bootClasspath = newWildcardEntry(jreLibPath)

   // jre/lib/ext/*
   jreExtPath := filepath.Join(jreDir, "lib", "ext", "*")
   self.extClassPath = newWildcardEntry(jreExtPath)
}

func getJreDir(option string) string {
   if option != "" && exists(option) {
      return option
   }
   if exists("./jre") {
      return "./jre"
   }
   if jh := os.Getenv("JAVA_HOME"); jh != "" {
      return filepath.Join("jh", "jre")
   }
   panic("Can not find jre folder!")
}

// 判断目录是否存在
func exists(path string) bool {
   if _, err := os.Stat(path); err != nil {
      if os.IsNotExist(err) {
         return false
      }
   }
   return true

}

func (self *ClassPath) parseUserClasspath(option string) {
   if option == "" {
      option = "."
   }
   self.userClasspath = newEntry(option)
}
```

go语言结构体不需要显示实现接口，只要方法匹配即可，Go没有专门的构造函数。

其中：

```go
// jre/lib/*
jreLibPath := filepath.Join(jreDir, "lib", "*")
self.bootClasspath = newWildcardEntry(jreLibPath)

// jre/lib/ext/*
jreExtPath := filepath.Join(jreDir, "lib", "ext", "*")
self.extClassPath = newWildcardEntry(jreExtPath)
```

`newWildcardEntry`新建一个通配符entry。

entry_wildcard.go:

```go
// 通配符类路径不能递归匹配子目录下的JAR文件
func newWildcardEntry(path string) CompositeEntry {
	baseDir:=path[:len(path)-1] //去掉*
	compositeEntry := []Entry{}
	walkFn:= func(path string,info os.FileInfo,err error) error {
		if err!=nil {
			return err
		}
		if info.IsDir() && path !=baseDir {
			return filepath.SkipDir
		}
		if strings.HasSuffix(path,".jar") || strings.HasSuffix(path,".JAR") {
			jarEntry := newZipEntry(path)
			compositeEntry = append(compositeEntry,jarEntry)
		}
		return nil
	}
	err := filepath.Walk(baseDir, walkFn)
	if err != nil {
		return nil
	}
	return compositeEntry
}
```

> JAR 文件格式以流行的 ZIP 文件格式为基础。JAR 格式允许您压缩文件以提高存储效率。与 ZIP 文件不同的是，JAR 文件不仅用于压缩和发布，而且还用于部署和封装库、组件和插件程序，并可被像编译器和 JVM 这样的工具直接使用。在 JAR 中包含特殊的文件，如 manifests 和部署描述符，用来指示工具如何处理特定的 JAR。

其中，`newZipEntry`用来新建jar包实体。

entry_zip.go

```go
package classpath

import (
   "archive/zip"
   "errors"
   "io/ioutil"
   "path/filepath"
)

type ZipEntry struct {
   absPath string
}

func newZipEntry(path string) *ZipEntry {
   absPath, err := filepath.Abs(path)
   if err != nil {
      panic(err)
   }
   return &ZipEntry{absPath: absPath}
}

// 返回目录路径
func (self *ZipEntry) String() string {
   return self.absPath
}

func (self *ZipEntry) readClass(className string) ([]byte,Entry,error) {
   // 下图
}
```

![image-20211013124356224](https://gitee.com/hqinglau/img/raw/master/img/20211013124403.png)

在扫描用户路径时，会根据后缀新建composite、wildcard、zipentry或者direntry。

其中，entry_composite.go:

```go
package classpath

import (
	"errors"
	"strings"
)

type CompositeEntry []Entry

func newCompositeEntry(pathList string) CompositeEntry {
	compositeEntry := []Entry{}
	for _, path := range strings.Split(pathList, pathListSeparator) {
		entry := newEntry(path)
		compositeEntry = append(compositeEntry, entry)
	}
	return compositeEntry
}

func (c CompositeEntry) readClass(className string) ([]byte, Entry, error) {
	for _, entry := range c {
		data, from, err := entry.readClass(className)
		if err == nil {
			return data, from, nil
		}
	}
	return nil, nil, errors.New("class not found: " + className)
}

func (c CompositeEntry) String() string {
	strs := make([]string, len(c))
	for i, entry := range c {
		strs[i] = entry.String()
	}
	return strings.Join(strs, pathListSeparator)
}

```

对每个entry，单独调用。

给目录的话就直接根据目录和classname绝对路径读class。

entry_dir.go:

```go
package classpath

import (
   "io/ioutil"
   "path/filepath"
)

type DirEntry struct {
   absDir string // 绝对路径
}

// 返回目录路径
func (self *DirEntry) String() string {
   return self.absDir
}

// 创建结构体实例
func newDirEntry(path string) *DirEntry {
   absDir, err := filepath.Abs(path)
   if err != nil {
      panic(err)
   }
   return &DirEntry{absDir: absDir}
}

func (self *DirEntry) readClass(className string) ([]byte,Entry,error) {
   fileName := filepath.Join(self.absDir,className)
   data,err:=ioutil.ReadFile(fileName)
   return data,self,err

}
```

**总体图**：

![image-20211013131131079](https://gitee.com/hqinglau/img/raw/master/img/20211013131131.png)