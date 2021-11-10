url文件名和正经文件名对照：

```shell
pi@centos:~/www/md$ cat filename2name.txt 
test 测试用
linux_notes linux程序设计笔记
```

批量转换脚本：

```shell
#!/bin/bash
declare -A dimap;
eval `awk '{printf("dimap[%s]=%s;",$1,$2)}' filename2name.txt`
echo > tmp.mdfile
echo > ../index.html
for filename in $(ls);do 
	suffix=${filename##*.}; 
	if [ "$suffix" != "md" ]; then 
		continue;
	fi 
	name=${filename:0:${#filename}-3}
	marked -o ../blog/${name}.html < ${filename}
       	echo "${filename}:${dimap[${name}]}  to html done."
	echo -e "[${dimap[${name}]}](blog/${name}.html)\n" >> tmp.mdfile
done
echo "<h2>博客列表</h2><hr>" >> ../index.html
marked -o tmpindex.html < tmp.mdfile
cat tmpindex.html >> ../index.html
rm tmpindex.html
echo "index built done"
```

转换结果：

```shell
pi@centos:~/www/md$ bash md2html.sh 
linux_notes.md:linux程序设计笔记  to html done.
test.md:测试用  to html done.
index built done
```