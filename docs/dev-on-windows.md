# 在 Windows系统 下建立开发环境

## 基础

需要提前安装并配置好下列软件

- [`git`](https://git-scm.com/)  - 代码管理工具
- [`nginx`](http://nginx.org/en/download.html)  - web server
- [`postgresql`](https://www.postgresql.org/)  - 数据库 (建议安装10版本)
- [`node & npm`](https://nodejs.org/en/download/)  - npm 包管理工具
- [`python`](https://www.python.org/downloads/windows/) - 建议安装 3.7 版本。


## PostgreSQL

需要建立一个数据库，来存储 `camel-store` 的数据。

文档中约定使用的数据库名为 `camelstore`，其为用户 `camelstore` 所有。

> 建议使用 [`pgAdmin`](https://www.pgadmin.org/) 这个应用来完成新建数据库和用户的操作。

## git

1. 运行 `git clone https://github.com/gzqichang/camel-store.git --recurse-submodules` 把本仓库拉取到本地，记得需要加入 `--recurse-submodules` 参数。
1. 进入 `camel-store` 目录


## nginx

因为使用 `react.js` 编写管理后台，当调试 admin 的时候，需要 `nginx` 来分发静态资源请求和 api 请求。

1. 在`nginx`配置文件夹新增`camelstore.dev.com.nginx.conf`文件，写入以下内容：

```
server{
    listen 8080;
    server_name camelstore.dev.com;
    location / {
        proxy_pass $scheme://localhost:8001;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   Host              $http_host;
        proxy_set_header   X-Real-IP         $remote_addr;
    }
    location /api/ {
        proxy_pass $scheme://localhost:8000;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   Host              $http_host;
        proxy_set_header   X-Real-IP         $remote_addr;
    }
}
```
2. 用记事本打开 `nginx.conf` 文件，在 http { } 里面加入 `include camelstore.dev.com.conf;`，以引用 `camelstore.dev.com.conf` 文件里的配置。
3. 用 `nginx -t`  命令校验配置文件是否正确。
4. 运行 `nginx` 以启动nginx，如已启动，则重启 `nginx -s reload`。
5. 以管理员身份运行 `echo 127.0.0.1 camelstore.dev.com >> C:\Windows\System32\drivers\etc\hosts` 把域名加到 hosts 文件。

    1. 也可以管理员身份用记事本打开 `C:\Windows\System32\drivers\etc\hosts` , 末尾添加一行配置 `127.0.0.1 camelstore.dev.com`。

## api 部分

进入 `api` 目录。

1. 运行 `pip install -U pip setuptools pipenv` 安装升级包管理工具。

要安装各种依赖。

1. 运行 `pipenv sync` 安装虚拟环境。
1. 运行 `pipenv shell` 进入虚拟环境。
1. 运行一下 `django-admin version` 看看是不是 2.2 版本。目前我们还不支持 3.0 或更高版本。

`cd packages` 进入依赖包的目录，然后分别进入 `qapi`、`qcache`、`qsmstoken`、`quser` 目录，逐一运行 `python setup.py develop` 安装开发版本。

接下开始配置项目。

1. `cd conf/settings`
1. `copy local.py.tpl local.py` 生成本地配置环境。
1. 选择喜欢的编辑器，打开 `local.py` 文件，并把 `SECRET_KEY` 和 `DEFAULT_DB` 等项填好，记得 `SECRET_KEY` 要使用几十个字符的随机字符串。
1. 回退到 `api` 目录。
1. 运行 `python manage.py migrate` 创建各种数据表，然后运行
    1. `python manage.py init_staff` 创建初始用户数据
    1. `python manage.py format_groups` 创建初始分组数据
    1. `python manage.py updateconfig` 修改配置，相关的参数看一下帮助。
    1. `python manage.py wechatconfig` 修改配置，相关的参数看一下帮助。
    1. `python manage.py changepassword admin` 修改之前生成的 admin 账号的密码。
1. 运行 `python manage.py runserver` 跑起来看看。
    1. 打开浏览器，访问一下 `http://localhost:8000/api/sitemap/`, 如果可以看到 rest-framework 的界面，表示 api 已经正常运行。
    1. 再访问一下 `http://camelstore.dev.com:8080/api/sitemap/`, 理论上来说，应该可以看到同样的页面。如果不能访问，可能是 `nginx` 的监听端口不是 8080。

第三方配置，也是在 `local.py` 文件中，详见[第三方配置](third-party-config.md)。

## admin 部分

进入 `admin` 目录。

1. 运行 `npm install` 安装项目依赖的模块，这里视网络的情况，估计要一点时间。
1. 运行 `npm start` 把 admin 跑起来，
1. 打开浏览器，访问 `http://camelstore.dev.com:8080/` 应该可以看到 admin 的登陆界面，输入账号密码，可以成功登陆。

## wxapp 部分

进入 `wxapp` 目录。

1. 运行 `npm install` 安装项目依赖的模块，这里视网络的情况，估计要一点时间。
1. 运行 `npm i -g wepy-cli` 安排脚手架.
1. 运行 `npm run build` 编译 `wpy` 文件。
1. 用 `微信开发者工具` 打开项目 (在 `camel-store\wxapp\dist` 目录)，可以在模拟器中看到小程序界面。

默认数据接口是 `http://camelstore.dev.com:8080`，你可以在 `src/service/index.js` 文件中找到以下语句
```
export const baseUrl = 'http://camelstore.dev.com:8080';
```
修改 `baseUrl` 的值即可访问其他接口。


## 其它

> 推荐安装 [`fork`](https://git-fork.com/) 这个 git GUI 工具来完成其它日常操作，非常好用。

---------------

至此，开发环境建立完成。