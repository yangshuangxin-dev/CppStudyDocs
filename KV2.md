# NOSQL （KV存储）编译与测试用例

## 编译

一个项目工程，随着新增的文件越来越多，就需要编写makefile或者Cmake来进行对编译命令的简化。使用makefile或者Cmake可以更容易的进行代码编译，具体编写的方法可以参考链接： https://zhuanlan.zhihu.com/p/683841062

一个简单的KV存储Makefile编写如下所示：

```shell
# 定义基本变量
CC = gcc
FLAGS = -I ./epoll/include -L ./epoll/lib -lepoll
SRCS = kvstore.c  epoll_entry.c kvstore_array.c
TARGET = kvstore
SUBDIR = ./epoll/

OBJS = $(SRCS:.c=.o)
# 先编译子目录epoll，再编译KV store
all: $(SUBDIR) $(TARGET)
# 编译子目录
$(SUBDIR): ECHO
	make -C $@

ECHO:
	@echo $(SUBDIR)
# 编译目标kv store
$(TARGET): $(OBJS) 
	$(CC) -o $@ $^ $(FLAGS)

# 所有源文件都编译为obj文件
%.o: %.c
	$(CC) $(FLAGS) -c $^ -o $@
# 清理资源
clean: 
	rm -rf $(OBJS) $(TARGET)

```

## 测试用例

一个好的工程项目，需要使用测试用例保证功能的完整性、准确性、一致性。编写测试用例也是必不可少的一环，这里编写一个简单的测试用例，对KV store进行测试。

- 主函数，解析需要测试的ip、端口、方式（array: 0x01, rbtree: 0x02, hash: 0x04, skiptable: 0x08）

```c++
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <getopt.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <arpa/inet.h>
#define MAX_MAS_LENGTH		512
#define TIME_SUB_MS(tv1, tv2)  ((tv1.tv_sec - tv2.tv_sec) * 1000 + (tv1.tv_usec - tv2.tv_usec) / 1000)
int main(int argc, char *argv[]) {
	int ret = 0;
	char ip[16] = {0};
	int port = 0;
	int mode = 1;
	int opt;
	while ((opt = getopt(argc, argv, "s:p:m:?")) != -1) {
		switch (opt) {
			case 's':
				strcpy(ip, optarg);
				break;
			case 'p':
				port = atoi(optarg);
				break;
			case 'm':
				mode = atoi(optarg);
				break;
			default:
				return -1;
		}
	}
	// 连接服务端
	int connfd = ConnectTcpServer(ip, port);
	if (mode & 0x1) {
		// 统计耗时
		struct timeval tv_begin;
		gettimeofday(&tv_begin, nullptr);
		// 测试10w次数组存储
		ArrayTestCase10w(connfd);
		struct timeval tv_end;
		gettimeofday(&tv_end, nullptr);
		int timeUsed = TIME_SUB_MS(tv_end, tv_begin);
		printf("time_used: %d, qps: %d\n", time_used, 600000 * 1000 / timeUsed);
	}
}

```

- 连接KV存储服务器

```c++
int ConnectTcpServer(const char *ip, unsigned short port) {
	int connfd = socket(AF_INET, SOCK_STREAM, 0);
	struct sockaddr_in tcpserver_addr;
	memset(&tcpserver_addr, 0, sizeof(struct sockaddr_in));
	tcpserver_addr.sin_family = AF_INET;
	tcpserver_addr.sin_addr.s_addr = inet_addr(ip);
	tcpserver_addr.sin_port = htons(port);
	int ret = connect(connfd, (struct sockaddr*)&tcpserver_addr, sizeof(struct sockaddr_in));
	if (ret) {
		perror("connect");
		return -1;
	}
	return connfd;
}
```

- 发送TCP和接收TCP数据

```c++
int SendMsg(int connfd, char *msg, int length) {
	int res = send(connfd, msg, length, 0);
	if (res < 0) {
		perror("send");
		exit(1);
	}
	return res;
}	

int RecvMsg(int connfd, char *msg, int length) {
	int res = recv(connfd, msg, length, 0);
	if (res < 0) {
		perror("recv");
		exit(1);
	}
	return res;
}

```

- 比较测试用例的字符串是否一致

```c++
void equals(char *pattern, char *result, char *casename) {
	if (strcmp(pattern, result) == 0) {
		printf("==> PASS --> %s\n", casename);
	} else {
		printf("==> FAILED --> %s, '%s' != '%s'\n", casename, pattern, result);
	}
}
```

- 编写单个KV存储的测试用例

```c++
void TestCase(int connfd, char *msg, char *pattern, char *casename) {
	if (!msg||!pattern||!casename) {
        return;
    }
	SendMsg(connfd, msg, strlen(msg));
	char result[MAX_MAS_LENGTH] = {0};
	RecvMsg(connfd, result, MAX_MAS_LENGTH);
	equals(pattern, result, casename);
}

void ArrayTestcase(int connfd) {
	TestCase(connfd, "SET Name yangshuangxin", "SUCCESS", "SETCase");
	TestCase(connfd, "GET Name", "yangshuangxin", "GETCase");
	TestCase(connfd, "MOD Name YSX", "SUCCESS", "MODCase");
	TestCase(connfd, "GET Name", "YSX", "GETCase");
	TestCase(connfd, "DEL Name", "SUCCESS", "DELCase");
	TestCase(connfd, "GET Name", "NO EXIST", "GETCase");
}
```

- 编写压力测试，10w次测试

```c++
void ArrayTestCase10w(int connfd) { // 10w
	int count = 100000;
	int i = 0;
	while (i ++ < count) {
		ArrayTestcase(connfd);
	}
}
```
