上传按钮连点问题: 直接上传之后禁用，服务端也要做处理。

总体来说没啥，就是它的解析在post后面。

当HTTP POST传输multipart/form-data类型的数据时，请求头和请求体都需要遵循相应的格式要求。一个简单的multipart/form-data类型数据传输的报文如下：

```html
POST /blogUpload HTTP/1.1
Host: orzlinux.cn
Connection: keep-alive
Content-Length: 3846
sec-ch-ua: "Chromium";v="92", " Not A;Brand";v="99", "Google Chrome";v="92"
sec-ch-ua-mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryOUvxAG4zJFoW0Bm0
Accept: */*
Origin: null
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Accept-Encoding: gzip, deflate, br
Accept-Language: zh,zh-CN;q=0.9,en;q=0.8

------WebKitFormBoundaryOUvxAG4zJFoW0Bm0
Content-Disposition: form-data; name="file"; filename="filesystem20210830.md"
Content-Type: application/octet-stream

...这里应该是数据
------WebKitFormBoundaryOUvxAG4zJFoW0Bm0
Content-Disposition: form-data; name="filename"

filesystem20210830.md
------WebKitFormBoundaryOUvxAG4zJFoW0Bm0
Content-Disposition: form-data; name="passwd"

sdfasd
------WebKitFormBoundaryOUvxAG4zJFoW0Bm0--
```

根据解析，写了个上传博客的页面，保存之后`fork`然后`exec`生成网页和主页，代码我写的十分的`ugly`，可能还有一堆bug，再说。

![image-20210831153917749](https://gitee.com/hqinglau/img/raw/master/img/20210831153917.png)

如果这个文件能正常看的话，那就说明能跑起来。。。