---
title: jumpserver安装和使用详细文档
date: 2020-12-13 22:37:57
tags: []
published: true
hideInList: false
feature: /post-images/jumpserver-an-zhuang-he-shi-yong-xiang-xi-wen-dang.png
---
## 一 、 准备环境

- 硬件配置: 2个CPU核心, 4G 内存, 50G 硬盘（最低）
- 操作系统: Linux 发行版 x86_64
- Python = 3.6.x
- Mysql Server ≥ 5.7
- Redis

### 安装工具

更换yum源

```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo && sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo && yum clean all
```

安装 epel-release 和更换源

```shell
yum install  epel-release
wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
```

安装 其他工具

```shell
##安装 wget
yum install -y wget 
 ##安装 git
yum install -y git
 ## 安装 gcc
 yum install -y gcc
```



### 1.安装mysql

```shell
#1.添加mysq57源
yum -y localinstall http://mirrors.ustc.edu.cn/mysql-repo/mysql57-community-release-el7.rpm
#2。安装mysql-server
yum install mysql-community-server mysql-community-devel -y
#3.启动
[root@localhost ~]# systemctl enable mysqld
[root@localhost ~]# systemctl start mysqld
##修改root密码
---查看root默认密码---
[root@localhost ~]# grep 'password' /var/log/mysqld.log 
---修改----
[root@localhost ~]# mysql -uroot -p'Jj(dkdwVI5ba'
mysql> alter user 'root'@'localhost' identified by 'Com.123456';
##-----为jumpserver创建数据库以及用户
mysql> create database jumpserver default charset 'utf8' collate 'utf8_bin';
mysql> grant all on jumpserver.* to 'jumpserver'@'127.0.0.1' identified by 'Com.123456';
mysql> grant all on jumpserver.* to 'jumpserver'@'%' identified by 'Com.123456';
mysql> flush privileges;
```

### 2.搭建Python环境

```shell
##--安装
yum install python36-devel python36 -y

--构建虚拟环境--
[root@localhost ~]# python3.6 -m venv /opt/py3
--进入环境--
[root@localhost ~]# source /opt/py3/bin/activate

```

### 3.获取jumpserver的项目代码

```shell
cd /opt && \
wget https://github.com/jumpserver/jumpserver/releases/download/v2.5.3/jumpserver-v2.5.3.tar.gz
--解压--
tar zxvf jumpserver-v2.5.3.tar.gz
--重命名--
 mv jumpserver-v2.5.3 jumpserver
 ###wget下载慢可用迅雷下载再上传
```

### 4.安装编译环境

```shell
##安装编译环境
/opt/jumpserver/requirements/
rpm_requirements.txt #rpm 环境
requirements.txt #python环境
-------安装--------
yum install -y $(cat rpm_requirements.txt)
pip install -r requirements.txt
##若无法下载就修改pip源 
vi ~/.pip/pip.conf
---pip源---
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
##也可用以下命令直接安装
pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/


```

### 5.修改配置文件

```shell
(py3) [root@localhost jumpserver]# cp config_example.yml config.yml
(py3) [root@localhost jumpserver]# vi config.yml
# $ cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 49;echo
SECRET_KEY: nBAHxvmIK7QDvDNk3R2Gqpa32t4Jd7SECD8a5MZzskF6jiKP9

# SECURITY WARNING: keep the bootstrap token used in production secret!
# 预共享Token coco和guacamole用来注册服务账号，不在使用原来的注册接受机制
BOOTSTRAP_TOKEN: Htry5PvLiG85OhxJ8Cli8K23eZoX1KwWN9Jbb7RkBO8smOuY9

# 使用Mysql作为数据库
DB_ENGINE: mysql
DB_HOST: 127.0.0.1
DB_PORT: 3306
DB_USER: jumpserver
DB_PASSWORD: Com.123456
DB_NAME: jumpserver

```

## 6.启动jumpserver



```shell
(py3) [root@localhost jumpserver]# ./jms start all -d
##出现如下信息成功启动
gunicorn is running: 3776
flower is running: 3787
daphne is running: 3791
celery_ansible is running: 3795
celery_default is running: 3810
celery_node_tree is running: 3816
celery_check_asset_perm_expired is running: 3828
celery_heavy_tasks is running: 3832
beat is running: 3837

```

## 7.部署koko、下载lina和luna和Nigix整合

```shell
##下载koko--建议使用迅雷下载要不然慢
wget https://github.com/jumpserver/koko/releases/download/v2.5.3/koko-v2.5.3-linux-amd64.tar.gz
##解压并改名
tar -xf koko-v2.5.3-linux-amd64.tar.gz
mv koko-v2.5.3-linux-amd64/ koko

##将koko/下的kubectl 移动到/usr/local/bin/
(py3) [root@localhost opt]# cd koko/
(py3) [root@localhost koko]# ls
config_example.yml  init-kubectl.sh  koko  kubectl  locale  static  templates
(py3) [root@localhost koko]# mv kubectl /usr/local/bin/

##下载jumpserver 的  kubectl 改成可执行放到/usr/local/bin/ 改名为 rawkubectl
wget https://download.jumpserver.org/public/kubectl.tar.gz
tar -xf kubectl.tar.gz
chmod 755 kubectl
mv kubectl /usr/local/bin/rawkubectl

##--- 拷贝一份配置文件
(py3) [root@localhost koko]# cp config_example.yml config.yml
(py3) [root@localhost koko]# vi config.yml 

# 请和jumpserver 配置文件中保持一致，注册完成后可以删除
BOOTSTRAP_TOKEN: Htry5PvLiG85OhxJ8Cli8K23eZoX1KwWN9Jbb7RkBO8smOuY9

##执行kok
./koko


## 安装nginx
yum install nginx

##下载lina组件
cd /opt
wget https://github.com/jumpserver/lina/releases/download/v2.5.3/lina-v2.5.3.tar.gz
(py3) [root@localhost opt]# tar -xf lina-v2.5.3.tar.gz
(py3) [root@localhost opt]# mv lina-v2.5.3 lina
(py3) [root@localhost opt]# chown -R nginx:nginx lina

##下载Luna
cd /opt
wget https://github.com/jumpserver/luna/releases/download/v2.5.3/luna-v2.5.3.tar.gz
tar -xf luna-v2.5.3.tar.gz
mv luna-v2.5.3 luna
chown -R nginx:nginx luna 

##配置整合 
#nginx修改配置文件
#如下
```

```shell
server {
    listen 80;

    client_max_body_size 100m;  # 录像及文件上传大小限制

    location /ui/ {
        try_files $uri / /index.html;
        alias /opt/lina/;
    }

    location /luna/ {
        try_files $uri / /index.html;
        alias /opt/luna/;  # luna 路径, 如果修改安装目录, 此处需要修改
    }

    location /media/ {
        add_header Content-Encoding gzip;
        root /opt/jumpserver/data/;  # 录像位置, 如果修改安装目录, 此处需要修改
    }

    location /static/ {
        root /opt/jumpserver/data/;  # 静态资源, 如果修改安装目录, 此处需要修改
    }

    location /koko/ {
        proxy_pass       http://localhost:5000;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /guacamole/ {
        proxy_pass       http://localhost:8081/;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /ws/ {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:8070;
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /core/ {
        proxy_pass http://localhost:8080;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        rewrite ^/(.*)$ /ui/$1 last;
    }
}
```

```shell
##启动nginx
(py3) [root@localhost opt]# systemctl enable nginx
(py3) [root@localhost opt]# systemctl start nginx

#nginx -t
#nginx -s reload

```



----------------------------

# web页面

## 1.创建jumpserver用户

- 用于连接jumpserver 识别用户

- jump用户为实际人员使用的账户

- 通过用户登录jumpserver 再根据授权的资产使用系统用户登录系统

  ![](http://changetm.oss-cn-beijing.aliyuncs.com/Course/Attachment/d218bb02-9018-4783-9638-0cc1ea432835.png)

  

## 2.创建管理用户

- 管理用户是资产（被控服务器）上的 root，或拥有 NOPASSWD: ALL sudo 权限的用户， 
- JumpServer 使用该用户来 `推送系统用户`、`获取资产硬件信息` 等。

![](http://changetm.oss-cn-beijing.aliyuncs.com/Course/Attachment/1cc04ca8-c0a7-430c-9bba-04aa4031011c.png)

## 创建系统用户

- 系统用户是 JumpServer 跳转登录资产时使用的用户，可以理解为登录资产用户

```
开启自动推送
- 如果支持可以自动推送不支持请手动配置
设置密码
- 配置当前系统用户的密码
```

## 创建资源和授权用户

- 添加资源
- 为用户和用户组授权资源

![](http://changetm.oss-cn-beijing.aliyuncs.com/Course/Attachment/be89cd2a-f0f7-4870-998f-52898cbf7a51.png)

![](http://changetm.oss-cn-beijing.aliyuncs.com/Course/Attachment/fcf6543e-dec2-45a4-a810-8b04dd959037.png)

## 成功连接

- web连接

![](http://changetm.oss-cn-beijing.aliyuncs.com/Course/Attachment/c4303339-a721-47b1-bb3e-4dcabaa2ecf0.png)

- xshel 连接jump

  ![](http://changetm.oss-cn-beijing.aliyuncs.com/Course/Attachment/a8a1eea3-e9ad-4119-9eb0-b5f227d87642.png)

## P查看资产

![](http://changetm.oss-cn-beijing.aliyuncs.com/Course/Attachment/baff7d49-a5fc-4345-80b0-74630869c307.png)

## 输入数字进行连接

![](http://changetm.oss-cn-beijing.aliyuncs.com/Course/Attachment/20409016-ad09-46d6-aa56-c48e88c8a790.png)



备注：
如何忘记了管理用户admin的密码，可以这样来修改：
[root@jumpserver ~]# cd /opt/jumpserver/apps/
(py3) [root@jumpserver apps]# python change password admin
也可以创建新管理用户：
(py3) [root@jumpserver apps]# python manage.py createsuperuser --username=admin2 --email=2345@qq.com