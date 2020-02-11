# 在 ubuntu 下建立开发环境

## 1 基础安装

#### 1.1 更新软件源

```
$ sudo apt-get update
```

#### 1.2 安装基础依赖
```
$ sudo apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev git libmysqlclient-dev
```

#### 1.3 安装nginx
```
$ sudo apt-get install nginx
```

####  1.4 安装supervisor
```
$ sudo apt-get install supervisor
$
```

####  1.5 安装npm
```
$ sudo apt install npm
# 升级npm为最新版本
```

#### 1.6 安装Python3.7
使用pyenv进行python版本的管理，使用pyenv-virtualenv创建对应的虚拟环境
```
# 安装 pyenvcd 
$ git clone https://github.com/pyenv/pyenv.git ~/.pyenv
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bash_profile
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(pyenv init -)"' >> ~/.bash_profile
$ source .bash_profile

# 安装 pyenv-virtualenv
$ git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv
$ echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bash_profile
$ source .bash_profile

# 安装python 3.7.2
$ pyenv install 3.7.2

# 创建虚拟环境
$ pyenv virtualenv 3.7.2 camel-store
```
 **注意: 如果使用了 zsh 或者其他主题，可以将 .bash_profile 更换为 .zshenv, 只要是用户登录就会运行的脚本就可以 **


## 2 数据库配置

数据库使用的是 PostgreSQL，只需配置 `HOST`, `PORT`, `NAME`, `USER`, `PASSWORD` 这几个变量就可以了。

```
DEFAULT_DB = {
        'ENGINE': 'django.db.backends.postgresql',  # PostgreSQL数据库
        'HOST': '',  # 数据库主机
        'PORT': '',
        'NAME': '',  # 数据库名
        'USER': '',  # 数据库用户
        'PASSWORD': '',  # 数据库用户密码
        "ATOMIC_REQUESTS": True
}
```

## 3 安装项目依赖库

建议将项目包放在用户目录下，如camel-store项目放在文件夹 `~/project/camel-store`。

```
# 拉取项目
$ git clone https://github.com/gzqichang/camel-store.git --recurse-submodules
# 把本仓库拉取到本地，记得需要加入 `--recurse-submodules` 参数。

# 进入项目
$ cd ~/project/camel-store/camel-store

# 部署配置文件
$ cd ~/project/camel-store/deploy
# !如项目路径非 /home/ubuntu/project/camel-store/ 修改相关配置文件

# 文件 uwsgi.ini
[uwsgi]
env = LC_ALL=zh_CN.UTF-8
uid = ubuntu
gid = ubuntu
master = True
vacuum = True
processes = 2
threads = 2
chmod-socket  = 666

chdir = /home/ubuntu/project/camel-store/camel-store/api/
wsgi-file = conf/wsgi.py
socket = /home/ubuntu/project/camel-store/deploy/camel-store.sock
home = /home/ubuntu/.pyenv/versions/3.7.2/envs/camel-store
pidfile = /home/ubuntu/project/camel-store/deploy/camel-store.pid
py-autoreload = 1

;log-maxsize = 50000000  # 50M
;max-requests = 1000
;socket-timeout = 120
;post-buffering = 100M
;harakiri = 1200
;buffer-size = 65535
;listen = 2048
;reload-mercy = 4
preload=True
enable-threads=True


# 文件 supervisor.conf
[program:hzyc]
command = /home/ubuntu/.pyenv/versions/camel-store/bin/uwsgi --ini /home/ubuntu/project/camel-store/deploy/uwsgi.ini
stopsignal=QUIT
autostart=true
autorestart=true
user = ubuntu
stdout_logfile=/var/log/supervisor/camelstore_access.log
stderr_logfile=/var/log/supervisor/camelstore_error.log

# 文件 nginx.conf
upstream camelstore{
    server unix:///home/ubuntu/project/camel-store/deploy/camel-store.sock; # for a file socket
}

server {
    listen      80;
    #listen 443 ssl;
    server_name camelstore.lst.gzqichang.com;
    #ssl on;
    #ssl_certificate     /home/ubuntu/ssl/hzyc.lzx.gzqichang.com.crt;
    #ssl_certificate_key /home/ubuntu/ssl/hzyc.lzx.gzqichang.com.key;
    #ssl_session_timeout 5m;
    #ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    #ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    #ssl_prefer_server_ciphers on;

    charset     utf-8;

    # max upload size
    client_max_body_size 50M;   # adjust to taste
    # gzip
    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 2;
    gzip_vary on;


    # Django media
    location  /api/media  {
        alias /home/ubuntu/project/camel-store/camel-store/api/media;
        expires 10d;
    }

    location /api/static {
        alias /home/ubuntu/project/camel-store/camel-store/api/staticfiles;
        expires 10d;
    }

    location /api {
        uwsgi_pass  camelstore;
        include     uwsgi_params;
    }

    location / {
        add_header Cache-Control no-cache;
        alias /home/ubuntu/project/camel-store/camel-store/admin/dist/;
        index index.html index.htm;
    }

    location ~ ^.+\.txt$ {
        root /home/ubuntu/txt/;
    }

}


# 软连接配置文件
# !并非覆盖原有的nginx.conf 和 supervisor.conf
# 此处命名只为方便判断具体配置文件
$ sudo ln -s /home/ubuntu/camel-store/deploy/nginx.conf /etc/nginx/sites-enabled/camel-store.conf
$ sudo ln -s /home/ubuntu/camel-store/deploy/supervisor.conf /etc/supervisor/conf.d/camel-store.conf


# 激活虚拟环境
# 自动激活的方法
# 在 /home/ubuntu/project/camel-store下创建文件 .python_version，然后在里面写上camel-store
# 然后每次进入/home/ubuntu/project/camel-store这个文件夹，都会自动激活虚拟环境

```

#### 3.1 api 部分

进入 `api` 目录: `$ cd ~/project/camel-store/camel-store/api`

1. 安装依赖库
运行`pipenv install`
运行`pipenv install uwsgi`
运行一下 `django-admin -v` 看看是不是 2.2 版本。目前我们还不支持 3.0 或更高版本。

2. 安装本地依赖库
`cd packages` 进入依赖包的目录，然后分别进入 `qapi`、`qcache`、`qsmstoken`、`quser` 目录，逐一运行 `python setup.py develop` 安装开发版本。

3.接下开始配置项目。

	1. `cd conf/settings`
	2. `cp local.py.tpl local.py` 生成本地配置环境。
	3. 选择喜欢的编辑器，打开 `local.py` 文件，并把 `SECRET_KEY` 和 `DEFAULT_DB` 等项填好，记得 `SECRET_KEY` 要使用几十个字符的随机字符串。
	4. 回退到 `api` 目录。
	5. 运行 `python3 manage.py migrate` 创建各种数据表，然后运行
	    1. `python3 manage.py init_staff` 创建初始用户数据
	    2. `python3 manage.py format_groups` 创建初始分组数据
	    3. `python3 manage.py updateconfig` 修改配置，相关的参数看一下帮助。
	    4. `python3 manage.py wechatconfig` 修改配置，相关的参数看一下帮助。
	    5. `python3 manage.py changepassword admin` 修改之前生成的 admin 账号的密码。

第三方配置，也是在 `local.py` 文件中，详见[第三方配置](third-party-config.md)。


#### 3.2 admin 部分

进入 `admin` 目录。

1. 运行 `npm install` 安装项目依赖的模块，这里视网络的情况，估计要一点时间。
2. 运行 `npm run build`，如果不成功，可以考虑在本地`npm run build`后将dist/文件夹放入admin/文件夹下。


至此，Ubuntu开发环境部署完成。