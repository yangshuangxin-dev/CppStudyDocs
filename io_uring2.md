# IO_URING对比EPOLL

## 对比测试的客户端的实现

​	对比测试io_uring和epoll的性能，需要使用TCP客户端，进行连接到io_uring和epoll的服务器，进行发送数据和接收数据。为了通用，可以在运行客户端时传入参数为： -s 连接的ip地址； -p 连接的端口；-t 连接的线程；-n 发送数据的次数。

- 主函数和解析传入的参数

    ```c++
    #include <stdio.h>
    #include <string.h>
    #include <sys/socket.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <sys/time.h>
    #include <pthread.h>
    #include <arpa/inet.h>
    // 入参保存的结构体，传递给每一个线程
    typedef struct test_context_s {
    	char serverip[16];
    	int port;
    	int threadnum;
    	int requestion;
    	int failed;
    } test_context_t;
    // 统计时间差值
    #define TIME_SUB_MS(tv1, tv2)  ((tv1.tv_sec - tv2.tv_sec) * 1000 + (tv1.tv_usec - tv2.tv_usec) / 1000)
    int main(int argc, char *argv[]) {
    	int ret = 0;
        // 解析的参数放入结构体
    	test_context_t ctx = {0};
    	int opt;
    	while ((opt = getopt(argc, argv, "s:p:t:n:?")) != -1) {
    		switch (opt) {
    			case 's':
    				printf("-s: %s\n", optarg);
    				strcpy(ctx.serverip, optarg);
    				break;
    			case 'p':
    				printf("-p: %s\n", optarg);
    				ctx.port = atoi(optarg);
    				break;
    			case 't':
    				printf("-t: %s\n", optarg);
    				ctx.threadnum = atoi(optarg);
    				break;
    			case 'n':
    				printf("-n: %s\n", optarg);
    				ctx.requestion = atoi(optarg);
    				break;
    			default:
    				return -1;
    		}
    	}
        // 创建线程数组
    	pthread_t *ptid = malloc(ctx.threadnum * sizeof(pthread_t));
    	int i = 0;
    	// 记录开始时间
    	struct timeval tv_begin;
    	gettimeofday(&tv_begin, nullptr);
        // 创建发送数据和接收数据的线程
    	for (i = 0;i < ctx.threadnum;i ++) {
    		pthread_create(&ptid[i], nullptr, test_qps_entry, &ctx);
    	}
    	
    	for (i = 0;i < ctx.threadnum;i ++) {
    		pthread_join(ptid[i], nullptr);
    	}
    	// 线程结束时间
    	struct timeval tv_end;
    	gettimeofday(&tv_end, nullptr);
    	// 统计结果,耗时和qps
    	int time_used = TIME_SUB_MS(tv_end, tv_begin);
    	printf("success: %d, failed: %d, time_used: %d, qps: %d\n", ctx.requestion-ctx.failed, 
    		ctx.failed, time_used, ctx.requestion * 1000 / time_used);
    clean: 
    	free(ptid);
    	return ret;
    }
    ```

- TCP客户端的连接

    ```c++
    int connect_tcpserver(const char *ip, unsigned short port) {
    	// 分配fd
    	int connfd = socket(AF_INET, SOCK_STREAM, 0);
    	// 设置ip和端口
    	struct sockaddr_in tcpserver_addr;
    	memset(&tcpserver_addr, 0, sizeof(struct sockaddr_in));
    	tcpserver_addr.sin_family = AF_INET;
    	tcpserver_addr.sin_addr.s_addr = inet_addr(ip);// 字符串ip地址转换int32_t的地址
    	tcpserver_addr.sin_port = htons(port);
    	// tcp连接
    	int ret = connect(connfd, (struct sockaddr*)&tcpserver_addr, sizeof(struct sockaddr_in));
    	if (ret) {
    		perror("connect");
    		return -1;
    	}
    	// 成功放回fd
    	return connfd;
    }
    ```

- 发送数据和接收数据，对echo的数据进行对比

    ```c++
    constexpr char* TEST_MESSAGE = "YangshuangxinYangshuangxinYangshuangxinYangshuangxinYangshuangxin";
    constexpr uint16_t RBUFFER_LENGTH = 2048;
    constexpr uint16_t WBUFFER_LENGT = 2048;
    int send_recv_tcppkt(int fd) {
        // 拼接数据进行发送
    	char wbuffer[WBUFFER_LENGTH] = {0};
    	int i = 0;
    	for (i = 0;i < 1;i ++) {
    		strcpy(wbuffer + i * strlen(TEST_MESSAGE), TEST_MESSAGE);
    	}
    	int res = send(fd, wbuffer, strlen(wbuffer), 0);
    	if (res < 0) {
    		exit(1);
    	}
    	// 读取数据
    	char rbuffer[RBUFFER_LENGTH] = {0};
    	res = recv(fd, rbuffer, RBUFFER_LENGTH, 0);
    	if (res <= 0) {
    		exit(1);
    	}
    	// 服务器echo的数据进行对比
    	if (strcmp(rbuffer, wbuffer) != 0) {
    		printf("failed: '%s' != '%s'\n", rbuffer, wbuffer);
    		return -1;
    	}
    	return 0;
    }
    ```

- 线程入口，测试发送和接受数据

    ```c++
    static void *test_qps_entry(void *arg) {
        // 解析传入的参数
    	test_context_t *pctx = (test_context_t*)arg;
    	// 进行tcp连接
    	int connfd = connect_tcpserver(pctx->serverip, pctx->port);
    	if (connfd < 0) {
    		printf("connect_tcpserver failed\n");
    		return nullptr;
    	}
    	// 每一个线程需要发送的次数
    	int count = pctx->requestion / pctx->threadnum;
    	int i = 0;
    	int res;
    	// 循环发送
    	while (i++ < count) {
    		res = send_recv_tcppkt(connfd);
    		if (res != 0) {
    			printf("send_recv_tcppkt failed\n");
    			pctx->failed ++;
    			continue;
    		}
    	}
    	return nullptr;
    }
    ```

> 如果发送的数据大于MTU，会发生TCP的粘包和拆包，直接使用strcmp 对比发送的数据和接受的数据，不会相同。粘包和拆包的解决方法有两种，其一为前面两个字节增加长度；其二为增加分隔符，如  "\r\n"。

## 对比测试的结果

使用客户端调整不同的参数，对io_uring实现的TCP服务器和epoll实现的TCP服务器进行性能的对比测试，客户端开启50个线程，发送100w次数据包进行分别测试，测试结果如下：

| 测试数据包大小 | io_uring             | epoll                |
| -------------- | -------------------- | -------------------- |
| 65 byte        | 7876 ms,  126980 qps | 8898 ms,  112384 qps |
| 128 byte       | 7570 ms,  132100 qps | 9087 ms,  110047 qps |
| 256 byte       | 7645 ms,  130804 qps | 8692 ms,  115048 qps |
| 512 byte       | 7780 ms,  128534 qps | 9359 ms,  106849 qps |

从对比数据看，同样的服务器echo数据，**io_uring 性能大约比epoll 高 10% 左右**。