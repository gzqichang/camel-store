# 在 Ubuntu18 下部署
## 1.基础安装
#### 1.1 更新ubuntu的软件源
新建ubuntu虚拟机用户可能需要先把 `/etc/apt/sources.list` 中的源文件替换成国内镜像源
```
$ sudo apt-get update    // 更新安装源（Source）
$ sudo apt-get upgrade  // 更新已安装的软件包
$ sudo apt-get dist-upgrade // 更新已安装的软件包（识别并处理依赖关系的改变）
```
#### 1.2 安装数据库postgresql
需要建立一个数据库来存储`camel-store`的数据。
文档中约定使用的数据库名为 `camelstore`，其为用户 `camelstore` 所有。
```
# 安装postgresql
$ sudo apt-get install postgresql

# 创建数据库
$ sudo -u postgres psql
# 修改postgres用户的密码
postgres=# ALTER USER postgres WITH PASSWORD 'POSTGRES_PASSWORD'; 

# 创建此项目的的数据库及其拥有者
postgres=# create user camelstore with password 'YOUR_PASSWORD';
CREATE ROLE
postgres=# create database camelstore owner camelstore;
CREATE DATABASE
```
#### 1.3 Python
`camel-store` 至少需要 Python 3.6 或更高的版本，目前使用较多的是 3.6/3.7，建议使用`pyenv`进行python版本的管理，使用`pyenv-virtualenv`创建对应的虚拟环境， 使用`pipenv`管理虚拟环境。
在此均以 3.7.4 为例。
```
# 安装 pyenv
$ git clone https://github.com/pyenv/pyenv.git ~/.pyenv
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bash_profile
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(pyenv init -)"' >> ~/.bash_profile
$ source .bash_profile

# 安装 pyenv-virtualenv
$ git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv
$ echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bash_profile
$ source .bash_profile

# 在安装python前需要安装的一些依赖包
$ sudo apt-get install libc6-dev gcc
$ sudo apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm

# 安装python 3.7.4
$ pyenv install 3.7.4

# 创建虚拟环境
$ pyenv virtualenv 3.7.4 camel-store

# 此时通过命令 pyenv versions 应当可以看到下载的python版本和根据该版本创建的虚拟环境

# 安装pipenv
$ sudo pip install pipenv
# 如果因为网络原因不能下载，可使用国内源如：
$ sudo pip install pipenv -i https://pypi.tuna.tsinghua.edu.cn/simple
```

## 2. 项目代码
#### 2.1 通过git拉取代码
```
# 安装git
$ sudo apt-get install git
```
建议在 `/home/ubuntu/project/camel-store/` 目录下，运行 `git clone https://github.com/gzqichang/camel-store.git --recurse-submodules` 把本仓库拉取到本地，记得需要加入 `--recurse-submodules` 参数。

#### 2.2 使项目自动激活虚拟环境
在 `/home/ubuntu/project/camel-store/` 目录下创建文件`.python-version`，在文件内写入在步骤1.3中建立的虚拟环境名称camel-store。每当进入此目录时，应当能发现都可以自动激活虚拟环境。

#### 2.3 api部分
进入api目录。
1. 安装依赖库：
    1. 运行 `pipenv sync`，安装虚拟环境。
    2. 运行`django-admin --version`查看当前django版本是否为2.2版本，目前项目尚未支持django 3.0及以上版本。
2. 安装本地依赖库：
    `cd packages` 进入依赖包的目录，然后分别进入 qapi、qcache、qsmstoken、quser 目录，逐一运行 `python3 setup.py develop` 安装开发版本。
3. 开始配置项目：
    1.  `cd conf/settings`
    2. `cp local.py.tpl local.py` 生成本地配置环境。
    3. 选择喜欢的编辑器，打开 `local.py` 文件，并把 `SECRET_KEY` 和 `DEFAULT_DB` 等项填好，记得 `SECRET_KEY` 要使用几十个字符的随机字符串。
    4. 回退到 `api` 目录。
    5. 运行`python3 manage.py check`测试代码是否有问题
    6. 运行 `python3 manage.py migrate` 创建各种数据表，然后运行
        1. `python3 manage.py init_staff` 创建初始用户数据
        2. `python3 manage.py format_groups` 创建初始分组数据
        3. `python3 manage.py updateconfig` 修改配置，相关的参数看一下帮助。
        4. `python3 manage.py wechatconfig` 修改配置，相关的参数看一下帮助。
        5. `python3 manage.py changepassword admin` 修改之前生成的 admin 账号的密码。
        6. 运行 `python3 manage.py runserver`，此时程序应当能跑起来。
4. 第三方配置，也是在 `local.py` 文件中，详见[第三方配置](third-party-config.md)。

#### 2.4 admin部分
1. 安装软件
```
# 安装node.js和npm
$ sudo apt-get install nodejs-legacy
$ sudo apt-get install npm
# 安装用于管理node.js的版本的模块n
$ sudo npm install -g n
# 通过n模块安装最新版node.js
$ sudo n latest
# 升级npm为最新版本
$ sudo npm install npm@latest -g
# 查看版本
$ sudo node -v
$ sudo npm -v
```
2. 进入admin目录。
    1. 运行`npm install`安装项目依赖的模块，这里视网络的情况，估计要一点时间。
    2. 运行`npm run build`

#### 2.5 部署文件
###### 2.5.1 创建配置文件
在 `/home/ubuntu/project/camel-store/` 目录下创建文件夹deploy，在deploy目录下放置3个文件 `nginx.conf`， `supervisor.conf`，`uwsgi.ini`。

如果文件所在目录与文档项目目录不一致，需要自行修改其中路径。
nginx.conf文件中的server_name 需要填入自己的SERVER_NAME或者HOST。

###### 2.5.2 软连接配置文件
此处并非覆盖原有的nginx.conf和supervisor.conf
此处命名只为方便判断具体配置文件
```
$ sudo ln -s /home/ubuntu/camel-store/deploy/nginx.conf /etc/nginx/sites-enabled/camel-store.conf
$ sudo ln -s /home/ubuntu/camel-store/deploy/supervisor.conf /etc/supervisor/conf.d/camel-store.conf
```
###### 2.5.3 启动nginx
```
# 安装nginx
$ sudo apt-get install nginx
```
1. 运行`nginx -t`校验配置文件是否正确。
2. 运行`sudo service nginx start`以启动nginx, 如已启动，重启nginx则运行`sudo nginx -s reload`

###### 2.5.4 校验uwsgi
在deploy目录下运行`uwsgi --ini uwsgi.ini`，如正常运行，可ctrl+c键退出。

###### 2.5.5 启动supervisor
用supervisor管理uwsgi进程。
```
# 安装supervisor
$ sudo apt-get install supervisor
```
运行`sudo supervisorctl reload`

****
至此，在ubuntu环境中基本部署完成。