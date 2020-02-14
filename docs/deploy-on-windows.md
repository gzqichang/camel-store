# 在 Windows Server 2012 R2 下部署

## 基础

需要提前安装并配置好下列软件

- [`git`](https://git-scm.com/)  - 代码管理工具
- [`postgresql`](https://www.postgresql.org/)  - 数据库 (建议安装10版本)
- [`node & npm`](https://nodejs.org/en/download/)  - npm 包管理工具
- [`python`](https://www.python.org/downloads/windows/) - 建议安装 3.7 版本
    1. 安装的时候，有两个地方要特别注意，一是要勾选 `Add to PATH`，二是一定要把它安装到没有空格的路径下，比如 `c:\python37`
- `IIS` - windows server 自带的服务器管理系统
- `wfastcgi` - `pip install wfastcgi`
    1. 再运行一下 `wfastcgi-enable ` 命令启用它即可，成功运行后，会输出一个目录，可以把它加到配置文件（api部分会用到）

## PostgreSQL

需要建立一个数据库，来存储 `camel-store` 的数据。

文档中约定使用的数据库名为 `camelstore`，其为用户 `camelstore` 所有。

Windows Server 2012 与普通 windows 版本最大的不同，在于其文件 / 目录控制权限更严。所以在新建一个数据库的数据存放目录，比如 c:\pgsql\data, 右键点击文件夹，选择“属性”、“安全”、“编辑”、“Users”，把“完全控制”一行的“允许”选中。确认保存。

## git

1. 建议在 `C:\inetpub\wwwroot` 目录下，运行 `git clone https://github.com/gzqichang/camel-store.git --recurse-submodules` 把本仓库拉取到本地，记得需要加入 `--recurse-submodules` 参数。
1. 进入 `camel-store` 目录。

## api 部分

进入 `api` 目录，安装各种依赖。

1. 运行一下 `pip install -r .\requirements.txt`
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

    1. 在 `camel-store\api\conf\settings` 目录下，打开 `local.py` 文件，大概在 88 行看到下面内容：

```
STATIC_URL = '/api/static/'
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, "staticfiles/"),
]
```

回车下一行添加以下内容：

```
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
```

运行 `python manager.py collectstatic` 后把 `STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')` 删除后保存文件。
    
在 `api` 目录下，新建一个文本文件 `web.config`，写入以下内容：

```
<?xml version="1.0" encoding="UTF-8"?>
    <configuration>
        <system.webServer>
            <handlers>
                <add name="Python FastCGI" path="*" verb="*" modules="FastCgiModule" scriptProcessor="c:\python37\python.exe|c:\python37\lib\site-packages\wfastcgi.py" resourceType="Unspecified" requireAccess="Script" />
            </handlers>
        <rewrite>
            <rules>
                <rule name="static">
                    <match url="^api/static(.*).png$" />
                    <conditions>
                        <add input="{URL}" pattern="^api/static(.*).png$" negate="true" />
                    </conditions>
                    <action type="Rewrite" url="http://127.0.0.1:8000/{R:0}" />
                    <serverVariables>
                    </serverVariables>
                </rule>
            </rules>
        </rewrite>
        <directoryBrowse enabled="false" showFlags="Date, Time, Size, Extension" />
        </system.webServer>
        <appSettings>
            <add key="WSGI_HANDLER" value="django.core.wsgi.get_wsgi_application()" />
            <add key="PYTHONPATH" value="C:\inetpub\wwwroot\camel-store\api" />
            <add key="DJANGO_SETTINGS_MODULE" value="conf.settings.local" />
        <add key="WSGI_LOG" value="C:\Logs\camel-store-api.log" />
        <add key="WSGI_RESTART_FILE_REGEX" value=".*((\.py)|(\.config))$" />
        </appSettings>
    </configuration>
```
    
1. `scriptProcessor` 的值，要改为前文说过的运行 `wfastcgi` 输出的那个值。
1. `PYTHONPATH` 的 `value` 要改为 `manage.py` 的那个目录。
1. `WSGI_LOG` 的 `value` 改为存放log日志信息的目录路径。`注意`：需要右键点击log文件，选择“属性”、“安全”、“编辑”、“IIS_IUSRS”，把“完全控制”一行的“允许”选中。确认保存。如无 `IIS_IUSRS`，请自行添加。

另外，为了让静态文件的处理不经过Python这一层，建议往 `staticfiles` 和 `media` 目录下各放一个 `web.config` 文件，内容都是：

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <remove name="Python FastCGI" />
    </handlers>
  </system.webServer>
</configuration>
```

如果改过前面的 `web.config` 文件中的 `name` 值，这里也要对应。这样就可以在获取静态文件的时候快一点了。

## admin 部分

进入 `admin` 目录。

1. 运行 `npm install` 安装项目依赖的模块，这里视网络的情况，估计要一点时间。
1. 运行 `npm run build`。

进入 `admin\dist` 目录， 新建一个文本文件 `web.config`，写入以下内容：

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <defaultDocument>
            <files>
                <add value="admin/dist/index.html" />
            </files>
        </defaultDocument>
        <rewrite>
            <outboundRules>
                <preConditions>
                    <preCondition name="ResponseIsHtml1">
                        <add input="{RESPONSE_CONTENT_TYPE}" pattern="^text/html" />
                    </preCondition>
                </preConditions>
            </outboundRules>
            <rules>
                <rule name="api" enabled="true">
                    <match url="^api(.*)" />
                    <serverVariables>
                        <set name="HTTP_X_FORWARDED_HOST" value="{HTTP_HOST}" />
                    </serverVariables>
                    <action type="Rewrite" url="http://127.0.0.1:8000/{R:0}" />
                    <conditions>
                    </conditions>
                </rule>
            </rules>
        </rewrite>
    </system.webServer>
</configuration>
```

## IIS

在 `IIS` 中新建2个网站。

1. 一个用 `wfastcgi` 跑 `api` 这个 python 应用，在 127.0.0.1:8000 监听，物理路径：`camel-store\api` 目录。
    1. 添加虚拟目录，物理路径：`camel-store\api\staticfiles` 目录。
    1. 打开浏览器，访问一下 `http://127.0.0.1:8000/api/sitemap/`， 如果可以看到 rest-framework 的界面，表示 api 已经正常运行。

1. 另一个就是做静态文件服务和转发对 api/ 的请求到 127.0.0.1:8000，在 *:80 监听，物理路径：`camel-store\admin\dist` 目录，把域名分配过去就好。
    1. 打开 `URL重写` 功能，右侧的 `操作` 下方点击打开 `查看服务器变量`，打开后在右侧的 `操作` 下方点击 `添加`，添加服务器变量，名称为：`HTTP_X_FORWARDED_HOST`。
    1. 打开浏览器，访问分配过去的域名应该可以看到 admin 的登陆界面，输入账号密码，可以成功登陆。

---------------

至此，部署完成。可以访问 `http://camel-store-win.gzqichang.com/`。