## 基础命令

### 1. 帮助命令
```bash
docker version

docker info

docker --help
```
### 2. 镜像相关命令
**列出主机的所有镜像**
```bash
docker images [OPTIONS]
# OPTIONS 说明：
	-a :列出本地所有的镜像（含中间映像层）
	-q :只显示镜像ID
	--digests :显示镜像的摘要信息
	--no-trunc :显示完整的镜像信息
```

**在官网查找镜像信息**
```bash
docker search [OPTIONS] [NAME] 
# OPTIONS 说明
	--no-trunc : 显示完整的镜像描述
	-s : 列出收藏数不小于指定值的镜像。
	--automated : 只列出 automated build类型的镜像；
```

**删除镜像**
```bash
docker rmi [OPTIONS] [IMAGE_NAME | IMAGE_ID]
# OPTIONS 说明
 -f : 强制删除（不需要停止已经启动的容器）
```

拓展：命令的组合使用，通过 **`docker rmi -f $(docker images -qa)`** ，将两个命令配合使用，可以达到删除全部镜像的效果（尽量别用）。

### 3. 容器相关命令
**新建并且启动容器**
```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
# OPTIONS 说明
	--name="容器新名字": 为容器指定一个名称
	-d: 后台运行容器，并返回容器ID，也即启动守护式容器
	-i：以交互模式运行容器，通常与 -t 同时使用
	-t：为容器重新分配一个伪输入终端，通常与 -i 同时使用
	-P: 随机端口映射
	-p: 指定端口映射，有以下四种格式
	      ip:hostPort:containerPort
	      ip::containerPort
	      hostPort:containerPort
	      containerPort
```

-p 选项的详细解释
1. **`ip:hostPort:containerPort`**
    - 格式：宿主机IP地址:宿主机端口:容器端口
    - 示例：**`192.168.1.100:8080:80`**，这将容器内部的80端口映射到宿主机的IP地址192.168.1.100的8080端口上。
2. **`ip::containerPort`**
    - 格式：宿主机IP地址::容器端口
    - 示例：**`192.168.1.100::80`**，这将容器内部的80端口映射到宿主机的IP地址192.168.1.100的任意可用端口上。
3. **`hostPort:containerPort`**
    - 格式：宿主机端口:容器端口
    - 示例：**`8080:80`**，这将容器内部的80端口映射到宿主机的8080端口上，不指定IP地址，通常映射到所有可用的网络接口。
4. **`containerPort`**
    - 格式：仅指定容器端口
    - 示例：**`80`**，这将容器内部的80端口映射到宿主机的一个随机可用端口上。
启动 Ubuntu 实例的方法： **`docker run -it ubuntu /bin/bash`**，如果不使用这种方法，容器会因为没有前台运行的内容而自我销毁。

---

**列出正在运行的容器**

```bash
docker ps [OPTIONS]
# OPRIONS 说明
	-a :列出当前所有正在运行的容器+历史上运行过的
	-l :显示最近创建的容器。
	-n：显示最近n个创建的容器。
	-q :静默模式，只显示容器编号。
	--no-trunc :不截断输出。
```

---

**容器的 启动、停止 和 重启**

```bash
# 容器的启动
docker start [CONTAINER_NAME | CONTAINER_ID]

# 容器的停止
docker stop [CONTAINER_NAME | CONTAINER_ID]
# 容器的强制停止
docker kill [CONTAINER_NAME | CONTAINER_ID]

# 重启容器
docker restart [CONTAINER_NAME | CONTAINER_ID]
```

---

**容器的删除**
```bash
docker rm [CONTAINER_NAME | CONTAINER_ID]
```
使用上面提到的组合命令可以达到删除所有停止的容器的效果： **`docker rm -f $(docker ps -a -q)`**

---
**查看容器内的细节**

```bash
# 查看容器内运行的进程
docker top [CONTAINER_ID]

# 查看容器内部细节
docker inspect [CONTAINER_ID]
```
---
**进入正在运行的容器并以命令行交互**
```bash
# 方式一
docker exec -it [CONTAINER_ID] /bin/bash

# 方式二
docker attach [CONTAINER_ID]
```
attach 直接进入容器启动命令的终端，不会启动新的进程，exec 是在容器中打开新的终端，并且可以启动新的进程；通过 exec 进入后使用 exit 退出后，容器内仍然有前台进程运行，容器不会被销毁。
## 数据卷相关命令
### 1. 数据卷挂载
```bash
# 创建数据卷
docker volume create

#查看所有数据卷
docker volume ls

# 删除指定数据卷
docker volume rm

# 查看某个数据卷的详情
docker volume inspect

# 清除数据卷
docker volume prune
```
在创建容器的时候，使用 `docker run -v 数据卷名:容器内目录`
### 2. 本地目录挂载

```bash
docker run -v [本地目录]:[容器内目录]
# 例如：docker run -v ./mysql:/var/lib/mysql
```

本地目录必须以 / 或者 ./ 开头，否则会被识别为数据卷而非本地目录

## 自定义镜像相关命令

### 1. Docker File 指令

| 指令 | 说明 |
| --- | --- |
| FROM | 指定基础镜像 |
| ENV | 设置环境变量，可以在后面的指令中使用 |
| COPY | 拷贝本地文件到镜像中的指定目录 |
| RUN | 执行 Linux 的 Shell 命令，一般是安装过程的命令 |
| EXPOSE | 指定容器运行的时候监听的端口，提供给镜像使用者当作参考 |
| ENTRYPOINT | 容器入口命令，镜像中应用的启动命令，容器运行时调用 |

```bash
docker build [OPTIONS] [DOCKER_FILE_POSITION]
```
[OPTIONS 说明](https://www.runoob.com/docker/docker-build-command.html)
## Docker 网络相关命令

| 命令                                              | 说明        |
| ----------------------------------------------- | --------- |
| docker network create [NEWWORK_NAME]            | 创建一个网络    |
| docker network ls                               | 查看所有网络    |
| docker netword rm [NEWWORK_NAME]                | 删除指定网络    |
| docker network prune                            | 清除未使用网络   |
| docker network connect [NEWWORK] [CONTAINER]    | 使指定容器加入网络 |
| docker network disconnect [NEWWORK] [CONTAINER] | 使指定容器离开网络 |
| docker network inspect [NEWWORK_NAME]           | 查看网络的详细信息 |
## 数据拷贝相关命令
```
docker cp 容器名:要拷贝的文件在容器里面的路径 要拷贝到宿主机的相应路径
```