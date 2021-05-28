# 学习笔记: Docker部署Django由浅入深系列(上)：单容器部署Django + Uwsgi
> 原文作者大江狗([原文链接](https://mp.weixin.qq.com/s?__biz=MjM5OTMyODA4Nw==&mid=2247484582&idx=1&sn=e5dfbccdf546feaaa07ab1f2d2e09f58&chksm=a73c649e904bed88d4198b446ef2a3ad2b5c339ab772394aa2f8149a5a82dde1b0f877906075&scene=21#wechat_redirect))，本文根据实战经历稍作修改

Django在生产环境的部署还是比较复杂的, 令很多新手望而生畏, 幸运的是使用Docker容器化技术可以大大简化我们Django在生产环境的部署。Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移动的镜像中，然后发布到任何流行的 Linux机器上。由于未来使用Docker部署Django是大势所趋且小编对网上Docker部署Django的教程不甚满意(坑比较多), 于是决定自己写篇原创教程由浅入深地总结下Docker部署Django的整个过程。由于本文很长，我们将会分三篇发表于公众号【Python Web与Django开发】，主要内容如下：
- 上篇：[使用docker部署Django + Uwsgi（单容器)](https://pages.github.com/)
- 中篇：[使用docker部署Django + Uwsgi  + Nginx (双容器)](https://pages.github.com/)
- 下篇：[使用docker-compose部署Django + Uwsgi + Nginx + MySQL + Redis (多容器组合)](https://pages.github.com/)

**注意：** 本文侧重于Docker技术在部署Django时的应用，而不是Docker基础教程。对Docker命令不熟悉的读者们建议先学习下Docker基础命令。

## 学前核心知识必读
在正式开始我们的Docker之旅前，我们需要了解4个核心知识点：

1. 在Docker与virtualenv或pipenv的区别
   - virtualenv或pipenv创建的虚拟环境只是隔离了一个python运行的虚拟环境，允许不同的项目使用不同版本的程序包，从而解决依赖性问题。Docker的每个容器更像一个小型的linux系统，可以有自己的IP地址，容器相互之前环境隔离地更彻底。我们不仅可以把python的第三方依赖包放在一个容器里，我们还可以把数据库比如MySQL或Redis也放在容器里，这是python虚拟环境做不到的。因此生产环境使用Docker部署Django时，你不再需要使用virtualenv或pipenv创建python虚拟环境。

2. 在Docker镜像与容器之前的关系
   - Docker容器是由docker镜像创建的运行实例。简单来说，镜像是文件，容器是进程。它们之前的关系如同Python的类与实例化对象之前的关系，一个镜像可以对应多个容器。

3. 使用Docker技术的基本流程
   - 我们首先要使用docker pull命令或Dockerfile文件构建docker镜像，再使用docker run命令创建容器，最后使用docker exec -it命令进入容器执行其它命令。

4. 宿主机和容器间的通信
   - 安装Docker的服务器就是宿主机，宿主机有固定的IP地址和完整的操作系统比如Centos或Ubuntu。前面已经提到过每个容器像一个极简的Linux系统，还可以有自己的IP地址(Docker分配的)。宿主机和容器之间是可以通过docker cp或目录挂载的方式通信的。

## Docker的安装
学习本教程前首先我们要安装Docker。菜鸟教程上总结了Docker在各个平台和系统上的安装，大家可以参考。这里总结了下Docker在阿里云Ubuntu系统上的安装过程。步骤看似很多且复杂，但大家只需要一步一步copy和paste命令就行了，整个安装过程很流畅。

```
# 以Ubuntu为例
 # Step 1: 移除之前docker版本并更新更新 apt 包索引
 sudo apt-get remove docker docker-engine docker.io
 sudo apt-get update
 
 # Step 2: 安装 apt 依赖包，用于通过HTTPS来获取仓库
 sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
 
 # Step 3: 添加 Docker 的官方 GPG 密钥
 curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
 
 # Step 4: 设置docker稳定版仓库，这里使用了阿里云仓库
 sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
 sudo apt-get update
 
 # Step 5: 安装免费的docker Community版本docker-ce
 sudo apt-get -y install docker-ce
 # sudo apt-get install -y docker-ce=<VERSION> #该命令可以选择docker-ce版本
 
 # Step 6: 查看docker版本及运行状态
 sudo docker -v
 sudo systemctl status docker
 
 # Step 7：本步非必需。使用阿里云设置Docker镜像加速，注意下面链接请使用阿里云给自己的URL
 sudo mkdir -p /etc/docker
 sudo tee /etc/docker/daemon.json <<-'EOF'
 {  "registry-mirrors": ["https://ua3456xxx.mirror.aliyuncs.com"] }
 EOF
 sudo systemctl daemon-reload
 sudo systemctl restart docker
```

## 部署一个最简单的Django项目
现在我们要在服务器上利用Docker部署下面一个最简单的Django项目。我们不使用uwsgi和nginx，数据库也使用默认的sqlite3，只把django放在一个容器里。整个项目结构如下所示，目前该项目放在宿主机(服务器)上。

```
mysite1
 ├── db.sqlite3
 ├── Dockerfile # 用于生产docker镜像的Dockerfile
 ├── manage.py
 ├── mysite1
 │   ├── asgi.py
 │   ├── __init__.py
 │   ├── settings.py
 │   ├── urls.py
 │   └── wsgi.py
 ├── pip.conf # 非必需。pypi源设置成国内，加速pip安装
 └── requirements.txt # 项目只依赖Django，所以里面只有django==3.0.5一条
```

注意：Django默认`ALLOWED_HOSTS = []`为空，在正式部署前你需要修改`settings.py`, 把它设置为服务器实际对外IP地址，否则后面部署会出现错误，这个与docker无关。即使你不用docker部署，`ALLOWED_HOSTS`也要设置好的。

### 第一步：编写Dockerfile，内容如下：
```
 # 建立 python3.7 环境
 FROM python:3.7
 
 # 镜像作者大江狗
 MAINTAINER DJG
 
 # 设置 python 环境变量
 ENV PYTHONUNBUFFERED 1
 
 # 设置pip源为国内源
 COPY pip.conf /root/.pip/pip.conf
 
 # 在容器内/var/www/html/下创建 mysite1文件夹
 RUN mkdir -p /var/www/html/mysite1
 
 # 设置容器内工作目录
 WORKDIR /var/www/html/mysite1
 
 # 将当前目录文件加入到容器工作目录中（. 表示当前宿主机目录）
 ADD . /var/www/html/mysite1
 
 # 利用 pip 安装依赖
 RUN pip install -r requirements.txt
 ```
 
 我们还将pip源设置成了阿里云镜像，`pip.conf`文件内容如下所示：
 ```
 [global]
 index-url = https://mirrors.aliyun.com/pypi/simple/
 [install]
 trusted-host=mirrors.aliyun.com
 ```
### 第二步：使用当前目录的 Dockerfile 创建镜像，标签为 django_docker_img:v1。
进入Dockerfile所在目录，输入如下命令：
 ```
 # 根据Dockerfile创建名为django_docker_img的镜像，版本v1，.代表当前目录
 sudo docker build -t django_docker_img:v1 .
 
 # 查看镜像是否创建成功, 后面-a可以查看所有本地的镜像
 sudo docker images
 ```
 这是你应该可以看到有一个名为django_docker_img的docker镜像创建成功了，版本v1。
![image](https://user-images.githubusercontent.com/58221386/120001690-54b43400-bfd4-11eb-8b27-7ce2703a0383.png)

### 第三步：根据镜像生成容器并运行，容器名为mysite1, 并将宿主机的80端口映射到容器的8000端口。
 ```
 # 如成功，根据镜像创建mysite1容器并运行。宿主机80：容器8000。-d表示后台运行。
 sudo docker run -it -d --name mysite1 -p 80:8000 django_docker_img:v1
 
 # 查看容器状态，后面加-a可以查看所有容器列表，包括停止运行的容器
 sudo docker ps
 
 # 进入容器，如果复制命令的话，结尾千万不能有空格。
 sudo docker exec -it mysite1 /bin/bash  
 ```
 这时你应该可以看到mysite1容器开始运行了。使用`sudo docker exec -it mysite1 /bin/bash`即可进入容器内部。
 ![image](https://user-images.githubusercontent.com/58221386/120001919-93e28500-bfd4-11eb-8554-419866c08013.png)
### 第四步：进入容器内部后，执行如下命令

 ```
 python3 manage.py makemigrations
 python3 manage.py migrate
 python3 manage.py runserver 0.0.0.0:8000
 ```
 
这时你打开Chrome浏览器输入http://your_server_ip，你就可以看到你的Django网站已经上线了，恭喜你！

## 从客户端到Docker容器内部到底发生了什么
前面我们已经提到如果不指定容器网络，Docker创建运行每个容器时会为每个容器分配一个IP地址。我们可以通过如下命令查看。
```
 sudo docker inspect mysite1 | grep "IPAddress"
```
![image](https://user-images.githubusercontent.com/58221386/120002658-592d1c80-bfd5-11eb-90c4-82c8746f55cc.png)

用户访问的是宿主机服务器地址，并不是我们容器的IP地址，那么用户是如何获取容器内部内容的呢？答案就是端口映射。因为我们将宿主机的80端口(HTTP协议)隐射到了容器的8000端口，所以当用户访问服务器IP地址80端口时自动转发到了容器的8000端口。

**注意：** 容器的IP地址很重要，以后要经常用到。同一宿主机上的不同容器之间可以通过容器的IP地址直接通信。一般容器的IP地址与容器名进行绑定，只要容器名不变，容器IP地址不变。

## 把UWSGI加入Django容器中的准备工作
在前面例子中我们使用了Django了自带的`runserver`命令启动了测试服务器，但实际生成环境中你应该需要使用支持高并发的uwsgi服务器来启动Django服务。尽管本节标题是把uwsgi加入到Django容器中，但本身这句话就是错的，因为我们Django的容器是根据`django_docker_img:v1`这个镜像生成的，我们的镜像里并没有包含uwsgi相关内容，只是把uwsgi.ini配置文件拷入到Django容器是不会工作的。

所以这里我们需要构建新的Dockerfile并构建新的镜像和容器。为了方便演示，我们创建了一个名为mysite2的项目，项目结构如下所示：
```
 mysite2
 ├── db.sqlite3
 ├── Dockerfile # 构建docker镜像所用到的文件
 ├── manage.py
 ├── mysite2
 │   ├── asgi.py
 │   ├── __init__.py
 │   ├── settings.py
 │   ├── urls.py
 │   └── wsgi.py
 ├── pip.conf
 ├── requirements.txt # 两个依赖：django==3.0.5 uwsgi==2.0.18
 ├── start.sh # 进入容器后需要执行的命令，后面会用到
 └── uwsgi.ini # uwsgi配置文件
```
新的Dockerfile内容如下所示：
```
 # 建立 python3.7 环境
 FROM python:3.7
 
 # 镜像作者大江狗
 MAINTAINER DJG
 
 # 设置 python 环境变量
 ENV PYTHONUNBUFFERED 1
 
 # 设置pypi源头为国内源
 COPY pip.conf /root/.pip/pip.conf
 
 # 在容器内/var/www/html/下创建 mysite2 文件夹
 RUN mkdir -p /var/www/html/mysite2
 
 # 设置容器内工作目录
 WORKDIR /var/www/html/mysite2
 
 # 将当前目录文件拷贝一份到工作目录中（. 表示当前目录）
 ADD . /var/www/html/mysite2
 
 # 利用 pip 安装依赖
 RUN pip install -r requirements.txt
 
 # Windows环境下编写的start.sh每行命令结尾有多余的\r字符，需移除。
 RUN sed -i 's/\r//' ./start.sh
 
 # 设置start.sh文件可执行权限
 RUN chmod +x ./start.sh
```
`start.sh`脚本文件内容如下所示。最重要的是最后一句，使用`uwsgi.ini`配置文件启动Django服务。
```
 #!/bin/bash
 # 从第一行到最后一行分别表示：
 # 1. 生成数据库迁移文件
 # 2. 根据数据库迁移文件来修改数据库
 # 3. 用 uwsgi启动 django 服务, 不再使用python manage.py runserver
 python manage.py makemigrations&&
 python manage.py migrate&&
 uwsgi --ini /var/www/html/mysite2/uwsgi.ini
 # python manage.py runserver 0.0.0.0:8000
```
`uwsgi.ni`配置文件内容如下所示。
```
 [uwsgi]
 project=mysite2
 uid=www-data
 gid=www-data
 base=/var/www/html
 
 chdir=%(base)/%(project)
 module=%(project).wsgi:application
 master=True
 processes=2
 
 http=0.0.0.0:8000 #这里直接使用uwsgi做web服务器，使用http。如果使用nginx，需要使用socket沟通。
 buffer-size=65536
 
 pidfile=/tmp/%(project)-master.pid
 vacuum=True
 max-requests=5000
 daemonize=/tmp/%(project)-uwsgi.log
 
 #设置一个请求的超时时间(秒)，如果一个请求超过了这个时间，则请求被丢弃
 harakiri=60
 #当一个请求被harakiri杀掉会，会输出一条日志
 harakiri-verbose=true
```
## 单容器部署 Django + UWSGI

**第一步：** 生成名为`django_uwsgi_img:v1`的镜像
```
 sudo docker build -t django_uwsgi_img:v1 .
```

**第二步：** 启动并运行mysite2的容器
```
 # 启动并运行mysite2的容器
 sudo docker run -it -d --name mysite2 -p 80:8000 django_uwsgi_img:v1
```

**第三步：** 进入mysite2的容器内部，并运行脚本命令start.sh
```
 # 进入容器，如果复制命令的话，结尾千万不能有空格。
 sudo docker exec -it mysite2 /bin/bash  
 # 执行脚本命令
 sh start.sh
```
以上两句命令也可以合并成一条命令
```
 sudo docker exec -it mysite2 /bin/bash start.sh
```
执行后效果如下所示。当你看到最后一句[uWSGI]时，说明uwsgi配置并启动完成。
![image](https://user-images.githubusercontent.com/58221386/120003774-55e66080-bfd6-11eb-98a0-060e9124f965.png)

这时你打开浏览器输入http://your_server_ip，你就可以看到你的Django网站已经上线了，恭喜你！这次是uwsgi启动的服务哦，因为你根本没输入`python manage.py runserver`命令。

**故障排查：** 此时如果你没有看到网站上线，主要有两个可能原因：

uwsgi配置文件错误。尤其http服务IP地址为0.0.0.0:8000，不应是服务器的ip:8000，因为我们uwsgi在容器里，并不在服务器上。

浏览器设置了http(端口80)到https(端口443)自动跳转,。因为容器8000端口映射的是宿主机80端口，如果请求来自宿主机的443端口，容器将接收不到外部请求。解决方案清空浏览器设置缓存或换个浏览器。

**注意：** 你会留意到网站虽然上线了，但有些图片和网页样式显示不对，这是因为uwsgi是处理动态请求的服务器，静态文件请求需要交给更专业的服务器比如Nginx处理。下篇文章中我们将介绍Docker双容器部署Django+Uwsgi+Nginx，欢迎关注。

小结
花时间学习使用Docker部署Django是非常值得的。因为构建的镜像是可以复用的，一但你熟悉了项目的布局和使用Docker部署和启动服务的基本流程，你会发现Django部署变得非常简单了，而且再也不用担心宿主机所在的操作系统或环境，真正实现了项目的可移植。
