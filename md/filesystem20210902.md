<style type="text/css"> img{display:block;margin:0 auto;}</style>

## 日志结构文件系统

> 内存不断增长，磁盘流量更多由写入组成，读取在缓存中处理。
>
> 随机IO与顺序IO性能差距。
>
> 现有文件系统在很多常见工作负载表现不佳，如 FFS 创建一个文件，却需要写入大量数据：inode，位图，目录，目录 inode。
>
> 文件系统不支持RAID。

LFS（Log-Structured File System)

写入磁盘时，先将所有更新，包括元数据，缓冲在内存段，内存段满时，顺序传输写入磁盘。不覆写现有数据，始终写入空闲位置。

如将两组更新缓冲，之后提交到磁盘：

![image-20210902124744859](https://gitee.com/hqinglau/img/raw/master/img/20210902124744.png)

问题：inode 在变，怎么查找？

inode 映射。在 inode 号和 inode 之间引入了间接层。imap，inode 号对应最新版的磁盘地址。imap 放哪？和数据、inode 放一起。

![image-20210902130052116](https://gitee.com/hqinglau/img/raw/master/img/20210902130052.png)

imap 分散了，怎么找 imap ？磁盘上固定位置，检查点区域（checkpoint region）。包含指向最新 inode 映射片段的地址，定期更新。

![image-20210902130551445](https://gitee.com/hqinglau/img/raw/master/img/20210902130551.png)

接下来考虑目录，目录也要访问更新。如在目录中创建文件 foo。

![image-20210902131802738](https://gitee.com/hqinglau/img/raw/master/img/20210902131802.png)

> 递归更新问题：更新 inode 时，它在磁盘上位置发生变化，导致对文件目录的更新，进一步更新目录的父目录。inode 映射避免了这个问题，inode 变化，更改并非反映在目录，而是 imap。

不断写入新位置，旧数据称为垃圾。

LFS 清理程序：定期读入 M 段，确定哪些块活着，活着的打包成 N 段，M 段释放，N 段写入新位置。

LFS 每个段段头部记录一个段摘要块，记录数据块的 inode 号及偏移量。对于一个块，查找段摘要块的 inode 号 N 和偏移量 T 。然后查 imap 找到 N 的位置，读取 N。利用 T 查数据块的真实位置，和我们要判断的数据块一样就是活的块。捷径：文件被截断或删除，在 imap 中记录一个版本号，比较磁盘段中的版本号，判断死活。

CR 更新：两个，交替更新，防止系统崩溃。

## 数据完整性和保护

常见的两种单块故障：潜在扇区错误 LSE（Latent Sector Erros）、block corruption。

由于磁头碰撞或宇宙射线，数据位不可读或不准确。LSE。

block corruption，如写入错误的位置或由于总线有问题，数据有误。

检测 block corruption：每个块计算校验和，常用循环冗余校验（CRC）。

错误位置写入检测：在校验和中添加物理标识。

丢失的写入：报告说写进去了但没有，校验和也是旧的。经典解决方法：写入后验证，I/O 翻倍。或在其他位置添加校验和。

校验和何时检查？大部分数据很少访问，将保持未检查状态。disk scrubbing，定期读取系统的每个块来检查，如每晚或每周。

时间开销可能比较明显。