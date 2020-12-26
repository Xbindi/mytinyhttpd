A mirror for tinyhttpd(Tinyhttpd非官方镜像,Fork自[sourceForge](https://sourceforge.net/projects/tiny-httpd/),仅供学习)


### 编译 
Compile for Linux
```
 To compile for Linux:
  1) Comment out the #include <pthread.h> line.
  2) Comment out the line that defines the variable newthread.
  3) Comment out the two lines that run pthread_create().
  4) Uncomment the line that runs accept_request().
  5) Remove -lsocket from the Makefile.
```
### 运行
+ htdocs目录下存放测试的网页静态文件和脚本文件，测试CGI时需要本机安装PERL，同时安装perl-cgi
+ index.html必须没有执行权限，否则看不到内容，并且会产生Program received signal SIGPIPE, Broken pipe，因为程序中如果有可执行权限会当cgi脚本处理。
+ color.cgi、check.cgi必须要有执行权限。并且脚本中头部的perl路径要对应本机安装路径，如/usr/bin/perl或者/usr/local/bin/perl
+ color.cgi是用perl编写，也可使用shell或者pyhton等脚本语言进行编写实现
+ 本工程源码中服务器端口被设置为5001，经过make生成可执行文件后，执行./httpd即可启动，然后在浏览器输入本机ip：5001即可看到index.html页面，程序运行成功

### 源码分析
**tinyhttp运作框架图**

<p><img src="https://images2017.cnblogs.com/blog/422590/201711/422590-20171110141226856-793077190.png">
</p>

**tinyhttp函数解析**

```c
void accept_request(int);//处理从套接字上监听到的一个 HTTP 请求
void bad_request(int);//返回给客户端这是个错误请求，400响应码
void cat(int, FILE *);//读取服务器上某个文件写到 socket 套接字
void cannot_execute(int);//处理发生在执行 cgi 程序时出现的错误
void error_die(const char *);//把错误信息写到 perror 并退出
void execute_cgi(int, const char *, const char *, const char *);//运行cgi脚本，这个非常重要，涉及动态解析
int get_line(int, char *, int);//读取套接字的一行，把回车换行等情况都统一为换行符结束。
void headers(int, const char *);//把 HTTP 响应的头部写到套接字
void not_found(int);//返回找不到请求文件
void serve_file(int, const char *);//调用 cat 把服务器文件内容返回给浏览器。
int startup(u_short *);//开启http服务，包括绑定端口，监听，开启线程处理链接
void unimplemented(int);//返回给浏览器表明收到的 HTTP 请求所用的 method 不被支持。
```

<p>&nbsp; &nbsp; &nbsp;建议源码阅读顺序： main -&gt; startup -&gt; accept_request -&gt; execute_cgi, 通晓主要工作流程后再仔细把每个函数的源码看一看。</p>


**tinyhttp工作流程**

<img src="https://pic3.zhimg.com/80/v2-a8a9ca946e018c18a9293806fca234ea_1440w.png">
<p><br>
</p>

<p>&nbsp; &nbsp; &nbsp;（1） 服务器启动，在指定端口或随机选取端口绑定 httpd 服务。</p>
<p>&nbsp; &nbsp; &nbsp;（2）收到一个 HTTP 请求时（其实就是 listen 的端口 accpet 的时候），派生一个线程运行 accept_request 函数。</p>
<p>&nbsp; &nbsp; &nbsp;（3）取出 HTTP 请求中的 method (GET 或 POST) 和 url,。对于 GET 方法，如果有携带参数，则 query_string 指针指向 url 中 ？ 后面的 GET 参数。</p>
<p>&nbsp; &nbsp; &nbsp;（4） &#26684;式化 url 到 path 数组，表示浏览器请求的服务器文件路径，在 tinyhttpd 中服务器文件是在 htdocs 文件夹下。当 url 以 / 结尾，或 url 是个目录，则默认在 path 中加上 index.html，表示访问主页。</p>
<p>&nbsp; &nbsp; &nbsp;（5）如果文件路径合法，对于无参数的 GET 请求，直接输出服务器文件到浏览器，即用 HTTP &#26684;式写到套接字上，跳到（10）。其他情况（带参数 GET，POST 方式，url 为可执行文件），则调用 excute_cgi 函数执行 cgi 脚本。</p>
<p>&nbsp; &nbsp; （6）读取整个 HTTP 请求并丢弃，如果是 POST 则找出 Content-Length. 把 HTTP 200 &nbsp;状态码写到套接字。</p>
<p>&nbsp; &nbsp; （7） 建立两个管道，cgi_input 和 cgi_output, 并 fork 一个进程。</p>
<p>&nbsp; &nbsp; （8） 在子进程中，把 STDOUT 重定向到 cgi_output 的写入端，把 STDIN 重定向到 cgi_input 的读取端，关闭 cgi_input 的写入端 和 cgi_output 的读取端，设置 request_method 的环境变量，GET 的话设置 query_string 的环境变量，POST 的话设置 content_length 的环境变量，这些环境变量都是为了给 cgi 脚本调用，接着用 execl 运行 cgi 程序。</p>
<p>&nbsp; &nbsp; （9） 在父进程中，关闭 cgi_input 的读取端 和 cgi_output 的写入端，如果 POST 的话，把 POST 数据写入 cgi_input，已被重定向到 STDIN，读取 cgi_output 的管道输出到客户端，该管道输入是 STDOUT。接着关闭所有管道，等待子进程结束。这一部分比较乱，见下图说明：</p>
<p><br>
</p>
<p><img src="http://img.blog.csdn.net/20141226173222750?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamNqYzkxOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" width="484" height="222" alt=""><br>
</p>
<p>图 1 &nbsp; &nbsp;管道初始状态</p>
<p><br>
</p>
<p><img src="http://img.blog.csdn.net/20141226161119981?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamNqYzkxOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center" alt=""></p>
<p> 图 2 &nbsp;管道最终状态&nbsp;</p>
<p><br>
</p>
<p>&nbsp; &nbsp; （10） 关闭与浏览器的连接，完成了一次 HTTP 请求与回应，因为 HTTP 是无连接的。</p>
<p><br>
</p>

### 参考链接

https://github.com/EZLippi/Tinyhttpd

https://github.com/cbsheng/tinyhttpd

https://github.com/nengm/Tinyhttpd

https://zhuanlan.zhihu.com/p/24941375