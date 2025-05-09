# Nginx的基本使用

Nginx是最常用的服务器负载均衡和方向代理的中间件。通常用于HTTP、FTP、SFTP协议的代理，Nginx统一对外提供一个IP和端口，转发到其他的服务进程、服务器上进行处理。

淘宝网页使用的Server是Tengine，也是基于Nginx的http服务程序，163、知乎等几乎所有的服务都会使用到Nginx，Nginx主要用途就是作为网关，对外提供服务。外部的请求通过路由器、交换机等网络设备来到Nginx的服务器上，Nginx把请求转发到其他的进程、服务器等进行处理，最后由Nginx返回请求的资源。

## Nginx的编译安装

对于开源软件来说，其基本的编译的方式有两种，有Cmake文件就直接`mkdir build && cd build && cmake .. && make`，另外一种有configure文件，使用`./configure && make`。Nginx的编译就是使用configure文件。

在编译Nginx时，除了需要Nginx的源码外，一般还需要openssl支持https、pcre支持正则表达式、zlib支持压缩和解压缩。编译时需要指定这三个开源的源码路径，作为Nginx的模块和Nginx一起编译。

```shell
./configure --prefix=/usr/local/nginx --with-http_realip_module --with-http_addition_module --with-http_ssl_module --with-http_gzip_static_module --with-http_secure_link_module --with-http_stub_status_module --with-stream --with-pcre=/home/yangshuangxin/share/nginx/pcre-8.41 --with-zlib=/home/yangshuangxin/share/nginx/zlib-1.2.11 --with-openssl=/home/yangshuangxin/share/nginx/openssl-1.1.0g --add-module=/home/yangshuangxin/share/nginx/nty_module
make -j && make install
```

编译安装后，就可以启动Nginx：

```shell
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
# 不重启Nginx改变了配置文件，可以直接重新加载配置文件
/usr/local/nginx/sbin/nginx -s reload
# 停止nginx
/usr/local/nginx/sbin/nginx -s stop
```

## Nginx的配置文件

Nginx的配置文件最重要的是events，如果配置文件没有该配置直接启动Nginx会报错`[emerg] no "events" section in configuration`。可以指定Nginx的工作进程的数量4，每一个进程最大的连接数，可以如下配置：

```shell
worker_processes 4;
events {
	worker_connections 1024;
}
```

Nginx可以配置多个协议对应多个服务，是一个多对多的结构，最常用的就是HTTP/HTTPS协议，如果在http协议上有多个服务部署在不同的端口上，其配置文件如下：

```shell
http {
	server {
		listen 9000;
		location / {
			root /home/yangshuangxin/share/nginx/html9000/;
		}	
	}
	server {
		listen 9001;
		location / {
            root /home/yangshuangxin/share/nginx/html9001/;
                }
	}
	server {
		listen 9002;
		location / {
            root /home/yangshuangxin/share/nginx/html9002/;
                }
	}
	server {
		listen 9003;
		location / {
            root /home/yangshuangxin/share/nginx/html9003/;
                }
	}
}
```

可以访问部署在不同的端口上服务，如上面的配置文件。当收到HTTP的请求后，Nginx会返回root路径下的index.html的文件给客户端。可以使用浏览器直接访问，也可以使用`telent 192.168.0.120  9000`进行连接测试。

如上所示，启动监听了9000、9001、9002、9003这4个端口，其监听是在Nginx的master进程，但是进行真正进行TCP连接的建立和消息处理的是worker process工作进程，Nginx是多进程进行数据处理。（可以使用strace 跟踪系统调用）

这是因为Nginx在主进程进行listen()，然后fork()出子进程后，在子进程进行epoll_wait，其操作如下所示：

```c++
#include <errno.h>
#include <fcntl.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#define PROCESS_NUM 10
#define MAXEVENTS 64

int SockCreatBind(char *port) {
  int sock_fd = socket(AF_INET, SOCK_STREAM, 0);
  struct sockaddr_in serveraddr;
  serveraddr.sin_family = AF_INET;
  serveraddr.sin_port = htons(atoi(port));
  serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);

  bind(sock_fd, (struct sockaddr *)&serveraddr, sizeof(serveraddr));
  return sock_fd;
}

int MakeNonblocking(int fd) {
  int val = fcntl(fd, F_GETFL);
  val |= O_NONBLOCK;
  if (fcntl(fd, F_SETFL, val) < 0) {
    perror("fcntl set");
    return -1;
  }
  return 0;
}

int main(int argc, char *argv[]) {
  int sock_fd, epoll_fd;
  struct epoll_event event;
  struct epoll_event *events;
  if (argc < 2) {
    printf("usage: [port] %s", argv[1]);
    exit(1);
  }
  if ((sock_fd = SockCreatBind(argv[1])) < 0) {
    perror("socket and bind");
    exit(1);
  }
  if (MakeNonblocking(sock_fd) < 0) {
    perror("make non blocking");
    exit(1);
  }
  if (listen(sock_fd, SOMAXCONN) < 0) {
    perror("listen");
    exit(1);
  }
  if ((epoll_fd = epoll_create(MAXEVENTS)) < 0) {
    perror("epoll_create");
    exit(1);
  }

  event.data.fd = sock_fd;
  event.events = EPOLLIN;
  if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock_fd, &event) < 0) {
    perror("epoll_ctl");
    exit(1);
  }
  events = calloc(MAXEVENTS, sizeof(event));
  int i;
  for (i = 0; i < PROCESS_NUM; ++i) {
    int pid = fork();
    if (pid == 0) {
      while (1) {
        int num, j;
        num = epoll_wait(epoll_fd, events, MAXEVENTS, -1);
        printf("process %d return from epoll_wait\n", getpid());
        sleep(2);
        for (i = 0; i < num; ++i) {
          if ((events[i].events & EPOLLERR) || (events[i].events & EPOLLHUP) ||
              (!(events[i].events & EPOLLIN))) {
            fprintf(stderr, "epoll error\n");
            close(events[i].data.fd);
            continue;
          } else if (sock_fd == events[i].data.fd) {
            struct sockaddr in_addr;
            socklen_t in_len = sizeof(in_addr);
            if (accept(sock_fd, &in_addr, &in_len) < 0) {
              printf("process %d accept failed!\n", getpid());
            } else {
              printf("process %d accept successful!\n", getpid());
            }
          }
        }
      }
    }
  }
  wait(0);
  free(events);
  close(sock_fd);
  return 0;
}
```

Nginx最重要的功能就是负载均衡和反向代理。负载均衡可以配置转发的权重，高性能的服务器可以处理更多的请求，性能较低的服务器可以较低的处理请求。方向代理可以把请求转发到不同的端口、不同的服务器上进行处理，其配置如下：

```shell
http {

	upstream backend {
		server 192.168.0.120:9002 weight=1;
		server 192.168.0.120:9003 weight=2;
	}
	server {
		listen 9000;
		location / {
			proxy_pass http://backend;
		}	
	}
	server {
		listen 9001;
		location / {
                        root /home/yangshuangxin/share/nginx/html9001/;
                }
	}
	server {
		listen 9002;
		location / {
                        root /home/yangshuangxin/share/nginx/html9002/;
                }
	}
	server {
		listen 9003;
		location / {
                        root /home/yangshuangxin/share/nginx/html9003/;
                }
	}
}
```

在Nginx读取配置时，按照关键词进行字符串切分，例如读取worker_processes时，其处理回调函数配置如下：

```c++
// src\event\ngx_event.c
static ngx_command_t  ngx_core_commands[] = {
// ....
   { ngx_string("worker_processes"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_TAKE1,
      ngx_set_worker_processes, // 回调函数
      0,
      0,
      NULL },
    // .......
}

static char *
ngx_set_worker_processes(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_str_t        *value;
    ngx_core_conf_t  *ccf;
	// conf全局配置参数
    ccf = (ngx_core_conf_t *) conf;

    if (ccf->worker_processes != NGX_CONF_UNSET) {
        return "is duplicate";
    }

    value = cf->args->elts;
	// 配置文件分割后的但是保存在cf->args中
    if (ngx_strcmp(value[1].data, "auto") == 0) {
        ccf->worker_processes = ngx_ncpu;
        return NGX_CONF_OK;
    }

    ccf->worker_processes = ngx_atoi(value[1].data, value[1].len);

    if (ccf->worker_processes == NGX_ERROR) {
        return "invalid value";
    }

    return NGX_CONF_OK;
}

// 在src\os\unix\ngx_process_cycle.c 中启动主进程循环，按照配置文件创建工作进程
ngx_start_worker_processes(cycle, ccf->worker_processes,
                               NGX_PROCESS_RESPAWN);
// src\os\unix\ngx_process_cycle.c 中，使用fork创建工作进程
static void
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    ngx_int_t  i;
    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start worker processes");
    for (i = 0; i < n; i++) {
        ngx_spawn_process(cycle, ngx_worker_process_cycle,
                          (void *) (intptr_t) i, "worker process", type);
        ngx_pass_open_channel(cycle);
    }
}
// 在src\os\unix\ngx_process_cycle.c 中进行子进程的循环处理连接请求
static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    ngx_int_t worker = (intptr_t) data;
    ngx_process = NGX_PROCESS_WORKER;
    ngx_worker = worker;
    ngx_worker_process_init(cycle, worker);
    ngx_setproctitle("worker process");
    for ( ;; ) {
        if (ngx_exiting) {
            if (ngx_event_no_timers_left() == NGX_OK) {
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
                ngx_worker_process_exit(cycle);
            }
        }

        ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "worker cycle");

        ngx_process_events_and_timers(cycle); // 进行epll_wait
        if (ngx_terminate) {
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
            ngx_worker_process_exit(cycle);
        }

        if (ngx_quit) {
            ngx_quit = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                          "gracefully shutting down");
            ngx_setproctitle("worker process is shutting down");
            if (!ngx_exiting) {
                ngx_exiting = 1;
                ngx_set_shutdown_timer(cycle);
                ngx_close_listening_sockets(cycle);
                ngx_close_idle_connections(cycle);
            }
        }
        if (ngx_reopen) {
            ngx_reopen = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
            ngx_reopen_files(cycle, -1);
        }
    }
}
```

