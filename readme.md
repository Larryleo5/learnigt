# 溯源项目环境准备文档
## 服务器分配
* 10.0.90.214
	* ytrace(go) 小程序服务端
	* yhtrace_java(java) 数据管理系统
* 10.0.90.215
	* mongodb   


## 基础环境准备

溯源项目依赖于docker运行，服务器需要事先准备docker环境，包括 docker-ce，以及docker-compose

### docker-ce安装过程

参考[链接](https://docs.docker.com/install/linux/docker-ce/centos/)

* 移除旧版本


	``` shell
	  sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
	```
	
* 安装 docker-ce

	* 设置依赖
	
		1. 安装必须的依赖包
	
			``` shell
			sudo yum install -y yum-utils \
		  device-mapper-persistent-data \
		  lvm2
		
		
			```
		
		2. 添加docker镜像库
		
			``` shell
			
			sudo yum-config-manager \
		    --add-repo \
		    https://download.docker.com/linux/centos/docker-ce.repo
		
			```

	* 安装 docker-ce 

		``` shell
		sudo yum install docker-ce
		```
		
	* 开启服务
	
		开启服务后，让docker跟随系统启动
		
		``` shell
		
		sudo sytemctl enable docker
		
		```
		
		启动服务
		
		``` shell
		sudo systemctl start docker
		```
		
* 安装 docker-compose

	项目启动容器依赖于docker-compose，需要额外安装docker-compose来支持项目的部署
	
	``` shell
	sudo curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

	```
	
	修改执行权限
	
	``` shell
	
	sudo chmod +x /usr/local/bin/docker-compose

	
	```
	
* 测试docker以及docker-compose

``` shell
docker -v
docker-compose -v

```	

* 基础镜像准备

溯源项目依赖于golang基础环境与MongoDB，以及Ubuntu基础镜像，部署项目前需要提前准备镜像，或者项目启动时进行拉取

* Ubuntu:18.04
* golang:1.10.3
* mongo:4.0


### Java环境准备
安装java环境实则是安装 openjdk，需要先更新一下服务器

``` shell
yum update -y

```
-y 指的是默认确认

更新之后安装 openjdk

``` shell

yum install java-1.8.0-openjdk

```

测试java安装情况

``` shell
java -version
```

### git安装
使用yum 安装即可

``` shell
	yum install -y git
```
查看是否安装成功
```
 	git version
```
### 安装maven

* 新建maven目录

```
	mkdir /usr/local/maven3.5.4
```

* 下载maven3.5.4并解压到自定义的maven'文件夹

```
	wget http://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz
	tar -zxvf apache-maven-3.5.4-bin.tar.gz /usr/local/maven3.5.4
```

* 配置maven环境变量
使用的root权限修改全局的环境变量

``` 
	vim /etc/profile 
```

* 在文件末尾添加

```
	M2_HOME=/usr/local/maven3.5.4
	export PATH=${M2_HOME}/bin:${PATH}
```

* 使配置生效

``` 
	source /etc/profile 
```
## 启动mongo服务
#### 启动MongoDB时，映射MongoDB数据到宿主机

参考[链接](https://docs.docker.com/samples/library/mongo/#where-to-store-data),注意，需要提前创建 `/data/mongo/db`

``` shell
 docker run --name some-mongo -v /data/mongo/db:/data/db -d mongo

```


#### 注意：服务器外挂的存储500G，映射到服务器内路径为 /data/ 如需进行数据存储，一定要将路径映射到该路径下

### mongo-docker镜像启动
```
	docker run --name sy-mongo -p 27017:27017 -v /data/mongo:/data/db -d mongo:4.0
```
### 迁移测试环境中的数据
#### 进入docker镜像
```
	docker exec -ti 容器ID bash
```
在mongo容器中，mongo的可执行脚本放在/usr/bin目录下，进入该目录
主要使用 

* mongodump
* mongorestore

#### 数据迁移
mongodump：
命令格式：`mongodump -h dbhost  -d dbname -o dbdirectory`

```
	mongodump -h 10.0.90.152:21117 -d ytrace -o ~/ytrace_mongo/ytrace
```

* -h:  mongodb所在服务器地址，例如127.0.0.1，也可以指定端口:127.0.0.1:8080 
* -d:  需要备份的数据库名称，例如：test_data
* -o: 备份的数据存放的位置，例如：/home/bak
* -u: 用户名称，使用权限验证的mongodb服务，需要指明导出账号
* -p：用户密码，使用权限验证的mongodb服务，需要指明导出账号密码

#### 数据恢复
mongorestore：
命令格式：`mongorestore -h dbhost -d dbname dbdireactory`

```
	mongorestore -h 127.0.0.1:27017 -d ytrace  ~/ytrace_mongo/ytrace
```

* -h:  mongodb所在服务器地址
* -d:  需要恢复备份的数据库名称，例如：test_data，可以跟原来备份的数据库名称不一样
* directoryperdb: 备份数据所在位置，例如：/home/bak/test
* -drop: 加上这个参数的时候，会在恢复数据之前删除当前数据；

## 启动java服务
* git clone项目

```
	git clone http://10.0.90.51/blockchain/yhtrace_java.git 
```

* maven编译springboot项目

进入项目根目录 

```
	mvn clean package
```

* 将编译后的jar包和DockerFile文件拷贝到自定义的目录

```
	mkdir ~/docker_java
	cp src/main/docker/Dockerfile ~/docker_java/
	cp target/docker_spring_boot.jar ~/docker_java/
```
* docker_java 目录结构：

	* docker_spring_boot.jar
	* DcokerFile

* 使用DockerFile编译Docker镜像

```
	docker build -t docker:0.1 .
```

* java-docker启动

```
	docker run -d -p 9080:9080 docker:0.1
```
