终于弄好了域名的东西，备份，SSL证书。看下效果：

![image-20210813200539764](https://gitee.com/hqinglau/img/raw/master/img/20210813200541.png)

这个小锁的标识比起之前url前面“不安全”清爽了许多。

总体来说这个博客界面我还是比较喜欢的，很简约，虽然就是个文件服务器的功能，后续应该会陆续加上分类，时间。

我用的是腾讯云的服务器，设置https的方法还是比较简单的。

申请证书：

![image-20210813201049018](https://gitee.com/hqinglau/img/raw/master/img/20210813201050.png)

这里的SSL证书我只能申请具体域名的，不能用通配符，就申请了两个。

因为我服务器是自己写的，还不能处理https，所以就在想办法看能不能直接https -> http这样访问，还真可以。然后就看到了腾讯云的**一键HTTPS**:

![image-20210813201428799](https://gitee.com/hqinglau/img/raw/master/img/20210813201429.png)

回源协议正是我目前需要的东西，腾讯云去和浏览器打交道，就能实现80端口到https的处理。

但是又看到：

![image-20210813201600896](https://gitee.com/hqinglau/img/raw/master/img/20210813201603.png)

那我过几个月岂不是要重新搞或者要付费？

想靠“免费用熟悉之后再付费”这个模式赚我的钱？想的美，我现在就给你。然后我找到了CDN的方法处理https。

这个要掏钱，不过比较便宜。免费6个月，之后一年20块。

![image-20210813202022669](https://gitee.com/hqinglau/img/raw/master/img/20210813202023.png)

这里有个小插曲，我CDN之后域名无法访问：

![image-20210813202150763](https://gitee.com/hqinglau/img/raw/master/img/20210813202151.png)

我想不通，啥叫有效服务，后来和技术支持人员咨询之后才明白：

![image-20210813202335184](https://gitee.com/hqinglau/img/raw/master/img/20210813202336.png)

改过之后，再在腾讯云的CDN控制台改一下http强制转https，这样就能统一用https了。真不戳啊。

写博客也比较轻松了。直接markdown写好：

![image-20210813203409290](https://gitee.com/hqinglau/img/raw/master/img/20210813203411.png)

然后直接xshell发送到服务器，调用一个脚本，结束。

```shell
pi@centos:~/www/md$ rz -E
rz waiting to receive.
pi@centos:~/www/md$ bash ./md2html.sh 
blog2https.md:http+域名+CDN实现https访问  to html done.
linux_notes.md:linux程序设计笔记  to html done.
md_generate_html.md:md批量转换为html  to html done.
test.md:测试用  to html done.
index built done
```

附上链接：[hqinglau的博客](https://orzlinux.cn/)


