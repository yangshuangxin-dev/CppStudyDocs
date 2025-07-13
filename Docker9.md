# Docker的容器编排

容器编排是指管理和协调容器化应用程序的技术。包括：将多个容器组合成一个整体应用，同时负 责管理它们的生命周期、资源分配、网络连接等方面。常见容器编排工具包括 Docker Swarm、 Kubernetes等。

## Docker Compose单节点容器编排

docker compose 包含两个概念：

1. 项目，项目包含多个服务
2. 服务，即运行中的容器

### Docker Compose 安装

```shell
# 先把docker-compose文件dump到当前目录
wget https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-linux-x86_64
# 然后拷贝到/usr/bin/
sudo cp -arf docker-compose-linux-x86_64  /usr/bin/docker-compose
sudo chmod +x /usr/bin/docker-compose

# 卸载
sudo rm /usr/bin/docker-compose
```

### Compose 的常用flags

- --ansi 控制何时打印ANSI控制字符（“从不”|“始终”|“自动”）（默认值为“自动”)
- --compatibility 在向后兼容模式下运行compose
- --dry-run 在演习模式下执行命令
- --env-file stringArray  指定环境变量文件
- -f, --file stringArray  指定compose的配置文件
- --parallel int 控制最大并行度，-1表示无限制（默认值-1）
- --profile stringArray 指定要启用的配置文件
- --progress string 设置进程输出类型（auto、tty、plain、quiet）（默认为“auto”）
- --project-directory string 指定项目目录（默认值：首次指定的Compose文件的路径
- -p, --project-name string 指定项目名称

> 当指定了 -f，-p等flags的情况下，后续的docker-compose命令行操作需要携带这些参数才能正确操作，所以建议不要显示的指定-f，-p等flags

Compose 的常用命令行

- build 构建或重新构建服务的镜像
- config 解析、解析并呈现规范格式的compose文件，即检查配置文件
- cp 在服务容器和本地文件系统之间复制文件/文件夹
- create 为服务创建容器
- down 停止并移除容器、网络
- events 从容器接收实时事件
- exec 在正在运行的容器中执行命令
- images 列出创建的容器使用的镜像
- kill 强制停止服务容器
- logs 查看容器的输出
- ls 列出正在运行的compose项目
- pause 暂停服务
- port 打印端口绑定的公共端口
- ps 列出容器
- pull 拉取服务镜像
- push 推送服务镜像
- restart 重启服务容器
- rm 删除已停止的服务容器
- run 对服务运行一次性命令
- start 启动服务
- stop 停止服务
- top 显示运行的进程
- unpause 取消暂停
- up 创建并启动容器
- version 显示docker compose 版本信息

## Docker Swarm 多节点容器编排

### Swarm集群管理常用命令

- ca 显示并轮换rootCA
- init 初始化一个swarm集群
- join 向swarm集群添加一个工作节点或者一个管理节点
- join-token 生成加入集群的令牌
- leave 使当前节点离开集群
- unlock 解锁swarm节点
- unlock-key 获取解锁节点的key
- update 更新swarm集群

### Swarm 的node管理常用命令

- demote 降级一个或多个管理节点，使其成为工作节点
- inspect 显示一个或多个节点的详细信息
- ls 列出集群中所有节点
- promote 升级一个或多个工作节点，使其成为管理节点
- ps 列出节点上正在运行的任务，默认为当前节点
- rm 从集群中移除一个或多个节点
- update 更新节点信息

### Swarm 的Service管理常用命令

- create 创建服务
- inspect 显示一个或多个服务详情
- logs 查服务中任务的日志输出
- ls 列出所有服务
- ps 列出一个或多个服务的任务
- rm 删除一个或多个服务
- rollback 还原对服务配置的修改（回滚）
- scale 伸缩一个或多个replicated类型的服务
- update 更新服务信息

### Swarm 的Stack管理常用命令

- config 指定一个将要更新的配置文件，在进行合并和插值之后，输出最终的配置文件（用于检查）
- deploy 部署一个新的或者更新一个已存在的stack
- ls  列出所有stack
- ps 列出stack中的任务
- rm 移除一个或多个stack
- services 列出stack中的服务

