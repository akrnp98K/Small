---
title: Nginx解析漏洞深入利用及分析
top_img: https://www.apple.com.cn/newsroom/images/passions/education/Apple-STEAM-Day-hero_Full-Bleed-Image.jpg.large_2x.jpg
cover: https://www.apple.com.cn/newsroom/images/passions/education/Apple-STEAM-Day-hero_Full-Bleed-Image.jpg.large_2x.jpg
date: 2023-01-23 23:28:26
tags: 
      - 安全 
      - C++
      - 二进制安全
      - Service
categories: 
            - [安全]
            - [操作系统]
---

# 0x00 背景
Nginx历史上曾出现过多次解析漏洞，比如80sec发现的解析漏洞，以及后缀名后直接添加%00截断导致代码执行的解析漏洞。

但是在2013年底，nginx再次爆出漏洞（CVE-2013-4547），此漏洞可导致目录跨越及代码执行，其影响版本为：nginx 0.8.41 – 1.5.6，范围较广。

# 0x01 漏洞朔源
1.从官方补丁可以看出nginx在ngx_http_parse_request_line函数处做了代码patch，下载nginx源代码，定位其补丁文件为ngx_http_parse.c，函数ngx_http_parse_request_line中，分别位于代码段：

由此可定位本次漏洞需要分析的点，启用gdb调试，将break点设置为ngx_http_parse_request_line，

并且watch变量state和p，因为此函数为状态机，state为状态值，p为指针所指文志，这将是漏洞触发的关键点。

调试过程中需要跟踪nginx的worker子进程，所以需要设置setfollow-fork-mode child，并且在相应的地方设置断点，

![](https://image-1313245095.cos.ap-beijing.myqcloud.com/2023/01/23/16744869614887.jpg)
图-1 跟进子进程

2.分别发送正常和攻击语句进行测试：
正常语句：
``` shell
http://127.0.0.1/a.jpg
```
攻击语句：
``` shell
http://127.0.0.1/a.jpg（非编码空格）\0.php
```

使用正常语句一直s或n跟踪，会发现在对url的解析过程中，当路径中存在’.’或url存在’\0’会有如下处理：
``` cpp
#!cpp
case sw_check_uri:      
   ……
       case '.': 
           r->complex_uri = 1;  //此作为flag会判断使用ngx_http_parse_complex_uri方法，对路径修复
           state = sw_uri; 
           break;    
casesw_check_uri:    
   ……
        case '\0':   //当遇到\0是，将会判断为非法字符
           return NGX_HTTP_PARSE_INVALID_REQUEST;   
```

但是在检查uri中有空格则会进入到sw_check_uri_http_09的逻辑中，那么当我们发送攻击代码的时候，执行流程将如下：
![](https://image-1313245095.cos.ap-beijing.myqcloud.com/2023/01/23/16744870687358.jpg)
图-2 \0未触发异常

再回到sw_check_uri状态，此时后面的字符串为.php，而“.”将被为是uri的扩展名的分隔符
![](https://image-1313245095.cos.ap-beijing.myqcloud.com/2023/01/23/16744870895951.jpg)
图-3 取后缀名错误

最终导致nginx认为此次请求的后缀名为php，通过配置，会传给fastcgi进行处理，而fastcgi在查找文件的时候被\0截断，最终取到”a.jpg(非编码空格)”文件（注：Linux下php-fpm默认限制的后缀名为php，如未取消限制，访问将出现access denied。测试想要查看执行结果，需修改php-fpm.conf中的security.limit_extensions为空，即允许任意后缀名文件作为php解析。）

跨越
``` shell
location /protected / {deny all;}
```

的规则的原理与此类似，均为状态机中判断出现混乱，从导致而可以跨越到protected目录中，访问默认不可访问到的文件。

由此可知，常规利用中如果想触发代码执行，条件为可上传带空格的文件到服务器，并且服务器存储的时候也需要保留空格，而大多数情况下，web应用在处理上传文件时，都会将文件重命名，通过应用自身添加后缀，或者对后缀名去掉特殊字符后，做类型判断。

以上因素都导致此漏洞被认为是鸡肋漏洞，难以利用，而被人们所忽略。

# 0x02 windows下的RCE
此问题在windows的攻击场景中，则从小汽车变身为变形金刚。

首先，我们了解一下windows读取文件时的特点，即文件系统api创建文件名或查找文件名时，默认会去掉文件名后的空格，再执行操作，参见示例代码，目录下放置a.txt不带空格:

``` cpp
#!cpp
#include "stdafx.h"
#include<windows.h>

int _tmain(int argc, _TCHAR* argv[])
{
     HANDLE hFile =CreateFile(L"a.txt ",GENERIC_WRITE|GENERIC_READ, 0, //注意a.txt后有一个空格                    
              NULL,                  
              OPEN_EXISTING,          // 打开存在的文件
              FILE_ATTRIBUTE_NORMAL,   
              NULL);

     if (hFile ==INVALID_HANDLE_VALUE)
    {
      printf("openfailed!");
    }
     else
     {
      printf("fileopened");
     }

     CloseHandle(hFile);
     return 0;
}
```
通过此代码可知道，即使我们传入参数是”a.txt ”带空格，最后访问到却确是”a.txt”不带空格

此时的攻击过程为：
``` shell
1.上传任意文件（不需要带空格文件），
2.http://127.0.0.1/a.jpg(非编码空格)\0.php
```

![](https://image-1313245095.cos.ap-beijing.myqcloud.com/2023/01/23/16744875151368.jpg)
图-4文件a.jpg

![](https://image-1313245095.cos.ap-beijing.myqcloud.com/2023/01/23/16744875334728.jpg)
图-5漏洞利用

成功将a.jpg文件当作php代码执行，达到了攻击成功的目的。

通过windows的此特性，使CVE-2013-4547在windows+nginx的环境中的危害无限扩大，即在windows下，只要普通用户能上传文件，则可利用本次漏洞，导致代码执行，并进一步入侵服务器。