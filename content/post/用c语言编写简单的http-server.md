+++
title = "用c语言编写简单的http server"
date = "2015-06-06T19:55:21+08:00"
tags = ["c"]
author = "xiaojiong"

+++

#简单的http server

---

### 为什么要写这个？

> 最近看了很多关于c的书，一直想找一个机会实践下。正好自己也是一个web程序员，
之前对tcp的三次握手啊，http协议啊，理解的不是很好。于是心里就想写一个简单的http服务器，即能锻炼自己到c语言能力，又能帮助自己理解。

### 支持的功能

> * 简单的文本
> * 解析图片
> * cgi

### 实现分析
> 1. 解析浏览器的请求,在服务目录中查找相应的文件,如果找不到该文件就返回404错误页面
> 2. 检查是否为shell文件> 3. 如果该文件是shell文件:
	- 判断该文件是否有执行权限,如果没有权限发送403给客户端，否则
	- 发送HTTP/1.1 200 OK 给客户端	- 在子进程中popen该CGI程序
	- 得到CGI程序的结果文件指针
	- 将结果发送到客户端> 4. 如果该文件不可执行:	- 发送HTTP/1.1 200 OK 给客户端	- 如果是一个图片文件,根据图片的扩展名发送相应的Content-Type给客户端	- 如果不是图片文件,这里我们简化处理,都当作Content-Type: text/html	- 简单的HTTP协议头有这两行就足够了,再发一个空行表示结束	- 读取文件的内容发送到客户端
> 5. 关闭连接￼
### 源码
```
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <time.h>

#define MAXLEN 4096 //buf最大长度
#define PORT 8080   //端口号
#define DOCUMENT "/Users/xiaojiong/www" //根目录
#define INDEX "index.html" //默认文件
#define PROTOCOL "HTTP/1.0" //默认协议
#define SERVER "xiaojiong" //服务器名称 
#define RFC1123FMT "%a, %d %b %Y %H:%M:%S GMT" //RFC标准时间

char *get_mime_type( char* name );
int http_resonse(int client);
void send_error(FILE *wfd, int status, char *title, char *extra, char *text);
void send_file(FILE *wfd, char *path, struct stat *statbuf);
void exec_cgi(FILE *wfd, char *path, struct stat *statbuf);
void send_headers(FILE *wfd, int status, char *title, char *extra, char *mime, int length, time_t date);

/**
 * @synopsis 主函数， 实现socket和fork
 *
 * @param argc 标准输入 
 * @param argv 标准输入
 *
 * @return 0 
 */
int main(int argc, char** argv) {
    struct sockaddr_in stSockAddr;
    int SocketFD = socket(PF_INET, SOCK_STREAM, 0);
    pid_t pid;
     
    if(-1 == SocketFD)
    {
        perror("can not create socket");
        exit(EXIT_FAILURE);
    }

    memset(&stSockAddr, 0, sizeof(struct sockaddr_in));

    stSockAddr.sin_family = AF_INET;
    stSockAddr.sin_port = htons(PORT);
    stSockAddr.sin_addr.s_addr = INADDR_ANY;

    if(-1 == bind(SocketFD,(const struct sockaddr *)&stSockAddr, sizeof(struct sockaddr_in)))
    {
        perror("error bind failed");
        close(SocketFD);
        exit(EXIT_FAILURE);
    }

    if(-1 == listen(SocketFD, 10))
    {
        perror("error listen failed");
        close(SocketFD);
        exit(EXIT_FAILURE);
    } 
    while(1) {
        int client = accept(SocketFD, NULL, NULL);

        if(0 > client)
        {
            perror("error accept failed");
            close(SocketFD);
            exit(EXIT_FAILURE);
        }
        pid = fork();
        if (pid < 0) {
            perror("error fork failed");
            exit(EXIT_FAILURE);
        } else if(pid == 0) {
            http_resonse(client);
            close(client);
        } else {
            close(client);
            continue;
        }
    }
}
/**
 * @synopsis 处理http请求
 *
 * @param client 阻塞的客户端连接 
 *
 * @return 0 
 */
int http_resonse(int client) {
    int cgi = 0;
    char* dot;
    char *method;
    char *path;
    char *protocol;
    char buf[MAXLEN];
    char file[MAXLEN] = DOCUMENT;
    struct stat statbuf;
    FILE *rfd, *wfd;
    rfd = fdopen(client, "a+");   
    wfd = fdopen(dup(client), "w");

    //打印请求信息
    if (!fgets(buf, sizeof(buf), rfd)) 
        return -1;
    printf("URL: %s", buf);
    method = strtok(buf, " ");
    path = strtok(NULL, " ");
    protocol = strtok(NULL, "\r");
    if (!method || !path || !protocol) 
        return -1;
    strcat(file, path);
    if (strcasecmp(method, "GET") != 0) {
        send_error(wfd, 501, "Not supported", NULL, "Method is not supported.");
    } else if (stat(file, &statbuf) < 0) {
        send_error(wfd, 404, "Not Found", NULL, "File not found.");
    }
    //输出文件
    if (strcasecmp(path, "/") == 0) {
        strcat(file, INDEX);
    }
    //cgi
    dot = strrchr( file, '.' );
    if ( strcmp( dot, ".sh" ) == 0)
        cgi = 1;
    if (cgi) {
        exec_cgi(wfd, file, &statbuf);
    } else {
        send_file(wfd, file, &statbuf);
    }
    fclose(rfd);
    fclose(wfd);
    return 0;
}
/**
 * @synopsis 发送失败信息
 *
 * @param wfd 输出文件描述符
 * @param status http协议状态码
 * @param title 标题
 * @param extra http头的附加信息
 * @param text 内容
 */
void send_error(FILE *wfd, int status, char *title, char *extra, char *text) {
    send_headers(wfd, status, title, extra, "text/html", -1, -1);
    fprintf(wfd, "<HTML><HEAD><TITLE>%d %s</TITLE></HEAD>\r\n", status, title);
    fprintf(wfd, "<BODY><H4>%d %s</H4>\r\n", status, title);
    fprintf(wfd, "%s\r\n", text);
    fprintf(wfd, "</BODY></HTML>\r\n");
}
/**
 * @synopsis 发送文件内容给客户端(cgi)
 *
 * @param wfd 输出文件描述符
 * @param path 文件路径
 * @param statbuf 文件状态结构体
 */
void exec_cgi(FILE *wfd, char *path, struct stat *statbuf) {
    char data[MAXLEN];
    int n;
    //检查文件是否有可执行权限
    if (((statbuf->st_mode & S_IXUSR) != S_IXUSR) && ((statbuf->st_mode & S_IXGRP) != S_IXGRP) && ((statbuf->st_mode & S_IXOTH) != S_IXOTH)) {
        send_error(wfd, 403, "File not exec", NULL, "File not exec");
    } else {
        int length = -1;
        send_headers(wfd, 200, "OK", NULL, get_mime_type(path), length, statbuf->st_mtime);
        FILE *pp;
        pp = popen(path, "r");
        if (pp != NULL) {
            while ((n = fread(data, 1, sizeof(data), pp)) > 0)  {
                fwrite(data, 1, n, wfd);
            }
            pclose(pp);
        }
    }
}
/**
 * @synopsis 发送文件内容给客户端
 *
 * @param wfd 输出文件描述符
 * @param path 文件路径
 * @param statbuf 文件状态结构体
 */
void send_file(FILE *wfd, char *path, struct stat *statbuf) {
    char data[MAXLEN];
    int n;

    FILE *file = fopen(path, "r");
    if (!file) {
        send_error(wfd, 403, "Forbidden", NULL, "Access denied.");
    } else {
        int length = S_ISREG(statbuf->st_mode) ? statbuf->st_size : -1;
        send_headers(wfd, 200, "OK", NULL, get_mime_type(path), length, statbuf->st_mtime);
        while ((n = fread(data, 1, sizeof(data), file)) > 0)  {
            fwrite(data, 1, n, wfd);
        }
        fclose(file);
    }
}
/**
 * @synopsis 发送http响应头信息 
 *
 * @param wfd 输出文件描述符
 * @param status http协议状态码
 * @param title 标题
 * @param extra http头的附加信息
 * @param mime  Content-Type
 * @param length 响应体内容大小
 * @param date 时间
 */
void send_headers(FILE *wfd, int status, char *title, char *extra, char *mime, int length, time_t date) {
    time_t now;
    char timebuf[MAXLEN];

    fprintf(wfd, "%s %d %s\r\n", PROTOCOL, status, title);
    fprintf(wfd, "Server: %s\r\n", SERVER);
    now = time(NULL);
    strftime(timebuf, sizeof(timebuf), RFC1123FMT, gmtime(&now));
    fprintf(wfd, "Date: %s\r\n", timebuf);
    if (extra) fprintf(wfd, "%s\r\n", extra);
    if (mime) fprintf(wfd, "Content-Type: %s\r\n", mime);
    if (length >= 0) fprintf(wfd, "Content-Length: %d\r\n", length);
    if (date != -1) {
        strftime(timebuf, sizeof(timebuf), RFC1123FMT, gmtime(&date));
        fprintf(wfd, "Last-Modified: %s\r\n", timebuf);
    }
    fprintf(wfd, "Connection: close\r\n");
    fprintf(wfd, "\r\n");
}
/**
 * @synopsis 根据文件名获取http Content-Type
 *
 * @param name 文件名
 *
 * @return Content-Type 
 */
char *get_mime_type( char* name )
{
    char* dot;
    dot = strrchr( name, '.' );
    if ( dot == (char*) 0 )
        return "text/plain; charset=UTF-8";
    if ( strcmp( dot, ".html" ) == 0 || strcmp( dot, ".htm" ) == 0 || strcmp( dot, ".sh" ) == 0)
        return "text/html; charset=UTF-8";
    if ( strcmp( dot, ".xhtml" ) == 0 || strcmp( dot, ".xht" ) == 0 )
        return "application/xhtml+xml; charset=UTF-8";
    if ( strcmp( dot, ".jpg" ) == 0 || strcmp( dot, ".jpeg" ) == 0 || strcmp( dot, ".ico" ) == 0)
        return "image/jpeg";
    if ( strcmp( dot, ".gif" ) == 0 )
        return "image/gif";
    if ( strcmp( dot, ".png" ) == 0 )
        return "image/png";
    if ( strcmp( dot, ".css" ) == 0 )
        return "text/xml; charset=UTF-8";
    return "text/plain; charset=UTF-8";
}
```


