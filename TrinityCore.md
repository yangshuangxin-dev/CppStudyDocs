# TrinityCore的编译和安装

TrinityCore是魔兽世界的游戏服务器代码，是学习网络服务器开发非常好的框架代码。TrinityCore的编译有以下步骤。

## 安装依赖

Ubuntu环境上安装需要执行以下命令：

```shell
sudo apt-get update
sudo apt-get install git clang cmake make gcc g++ libmysqlclient-dev libssl-dev libbz2-dev libreadline-dev libncurses-dev libboost-all-dev mysql-server p7zip
sudo update-alternatives --install /usr/bin/cc cc /usr/bin/clang 100
sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/clang 100
```

> 使用到libboost库的boost.asio，windows上是异步iocp，Linux上是Reator，在linux上实际上是同步IO

Windows上的开发环境需要安装这些软件:

- Boost >= 1.73
- MySQL >= 5.7
- OpenSSL = 1.1.x
- CMake >= 3.18.4
- visual studio (Community) 2019及以上版本

Windows上安装上面的软件后需要使用Cmake配置生成项目文件:

1. 源码同级目录创建 build 目录
2. 运行 CMake GUI
3. 浏览源 -> 选择源目录
4. 浏览构建 -> 选择构建目录（选择刚创建的 build 目录）
5. 单击配置
6. 生成（在 build 目录中可以看到 TrinityCore.sln 文件）

最后使用visual studio 打开 TrinityCore.sln 文件进行编译。

## TrinityCore源码下载和编译

服务器端编译下载

```shell
mkdir game
cd game
git clone -b 3.3.5 https://github.com/TrinityCore/TrinityCore.git
# Ubuntu编译方式
cd TrinityCore
mkdir build
cmake ../ -DCMAKE_INSTALL_PREFIX=/home/yangshuangxin/game  -DCONF_DIR=/home/yangshuangxin/game/bin
 
make -j4 && make install
```

客户端下载

```shell
# 下载地址
https://www.wowdl.net/client/World-of-Warcraft3.3.5a.12340-zhCN
https://www.wowdl.net/zh-CN/index
```

下载客户端，解压后，在根目录创建一个魔兽登录器.bat 批处理，指定客户端连接的服务器地址，添加如下内容：

```shell
#if not exist "WTF" md "WTF"
echo set realmlist 127.0.0.1>realmlist.WTF
echo set realmlist 127.0.0.1>data/enGB/realmlist.WTF
echo set realmlist 127.0.0.1>data/zhcn/realmlist.WTF
start wow.exe
goto end
```

> 可以使用WDBX Editor读取客户端里dbc文件夹的配置数据进行查看。

## 生成数据

需要客户端的Data、Interface、Camreas、dbc、Buildings文件夹里的内容拷贝到编译TrinityCore的环境上，使用服务器端程序生成服务器端可用的资源数据。

```shell
cd ~
mkdir res
cd res

#dbc maps 地图数据
../game/bin/mapextractor
#vmaps 建筑物、山脉、水体、角色，怪物、npc
 mkdir vmaps
 ../game/bin/vmap4extractor
 ../game/bin/vmap4assembler Buildings vmaps
 # 地图移动数据、导航
 mkdir mmaps
 ../game/bin/mmaps_generator
```

生成数据库数据，创建数据库和数据表

```shell
 mysql -uroot -p123456
 source /home/mark/game/TrinityCore/sql/create/create_mysql.sql;
```

## 启动服务器

### authserver

```shell
# 开启一个会话
# 第一次启动将会失败，没关系，后面 worldserver 启动会加载数据库信息，之后能启动成功。
./authserver
```

### worldserver

1. 修改worldserver.conf 指定读取的资源数据路径`DataDir = "../../res"`
2. 下载 TDB_full_world_335.23011_2023_01_16.sql，拷贝到`~/game/bin`下
3. 修改 auth 库

```shell
mysql> use auth;
 # xx.xx.xx.xx 替换服务端的 ip 地址
 mysql> update realmlist set address = "xx.xx.xx.xx" where id = 1;
```

> 可以有多个worldserver，每一个worldserver都有一个唯一的id，在worldserver.conf的RealmID里配置。

启动服务器

```shell
# 开启另一个会话
./worldserver
```

最后可以使用客户端连接目标服务器，进行验证是否成功。

> 启动worldserver后可以使用GM命令，修改服务器内存，例如 创建账户`account create yangshuangxin yangshuangxin`，设置账户的GM权限：`account set gmlevel yangshuangxin 3 -1`，设置权限后可以在进入游戏后的chat聊天取直接调用GM命令。

## VSCODE 调试代码

可以使用vscode远程连接到编译服务器进行代码编译和调试，需要安装的插件有：

- Remode-ssh。进行远程的ssh连接
- clangd。进行代码的跳转。
- CMake。支持CMake语法。

> clangd还需要安装clangd的服务器。

然后在根目录下创建.vscode，文件夹里创建settings.json

```json
{ 
"cmake.buildDirectory": "${workspaceFolder}/build",
 "cmake.buildEnvironment": {"CMAKE_EXPORT_COMPILE_COMMANDS": "ON"},
 "clangd.arguments": [
 "--background-index",
 "--compile-commands-dir=${workspaceFolder}/build"
    ],
 "clangd.fallbackFlags": [
 "-I${workspaceFolder}/src/common/",
 "-I${workspaceFolder}/src/common/Utilities/"
    ],
 	"clangd.checkUpdates": true,
 	"clangd.path": "/home/yangshuangxin/clangd_17.0.3/bin/clangd"
 }
```

