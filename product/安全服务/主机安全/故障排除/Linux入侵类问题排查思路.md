
本文档将指导您如何排查 Linux 入侵类问题并提供被入侵后的安全优化建议。
>?若已明确入侵事件属于挖矿或木马，请按 [挖矿木马自助清理手册](https://cloud.tencent.com/developer/article/1834731) 进行处置。
>
## 深入分析，查找入侵原因
### 一、检查隐藏账户及弱口令
1. 检查服务器系统及应用账户是否存在**弱口令**：
 * 检查说明：检查管理员账户、数据库账户、MySQL 账户、tomcat 账户、网站后台管理员账户等密码设置是否较为简单，简单的密码很容易被黑客破解。
  * 解决方法：以管理员权限登录系统或应用程序后台，修改为复杂的密码。
   * 风险性：高。
2. 使用`last`命令查看下服务器近期登录的账户记录，确认是否有可疑 IP 登录过机器：
    * 检查说明：攻击者或者恶意软件往往会往系统中注入隐藏的系统账户实施提权或其他破坏性的攻击。
    * 解决方法：检查发现有可疑用户时，可使用命令`usermod -L 用户名`禁用用户或者使用命令`userdel -r 用户名`删除用户。
    * 风险性：高。
3. 通过`less /var/log/secure|grep 'Accepted'`命令或者`less /var/log/auth.log|grep 'Accepted'`命令，查看是否有可疑 IP 成功登录机器：
    * 检查说明：攻击者或者恶意软件往往会往系统中注入隐藏的系统账户实施提权或其他破坏性的攻击。
    * 解决方法：使用命令`usermod -L 用户名`禁用用户或者使用命令`userdel -r 用户名`删除用户。
    * 风险性：高。
4. 检查系统是否采用**默认管理端口**：
    * 检查系统所用的管理端口（SSH、FTP、MySQL、Redis 等）是否为默认端口，这些默认端口往往被容易自动化的工具进行暴破成功。
    * 解决方法：
        1. 在服务器内编辑`/etc/ssh/sshd_config`文件中的 Port 22，将22修改为非默认端口，修改之后需要重启 ssh 服务。
>!当对端口进行修改时，需同时在 [云服务器控制台](https://console.cloud.tencent.com/cvm/instance/index?rid=1) 上修改对应主机的安全组配置，在其入站规则中，放行对应端口，详情请参见 [添加安全组规则](https://cloud.tencent.com/document/product/215/39790)。
        2. 运行 `/etc/init.d/sshd restart`（Centos7以下）或 `systemctl restart sshd`（Centos7及其以上）或 `/etc/init.d/ssh restart`（Debian/Ubuntu）命令重启使配置生效。
        3. 修改 FTP、MySQL、Redis 等的程序配置文件的默认监听端口21、3306、6379为其他端口。
        4. 限制远程登录的 IP，编辑`/etc/hosts.deny` 、`/etc/hosts.allow`两个文件来限制 IP。
    * 风险性：高。
5. 检查`/etc/passwd`文件，看是否有非授权账户登录：
    * 检查说明：攻击者或者恶意软件往往会往系统中注入隐藏的系统账户实施提权或其他破坏性的攻击。
    * 解决方法：使用命令`usermod -L 用户名`禁用用户或者使用命令`userdel -r 用户名`删除用户。
    * 风险性：中。

### 二、检查恶意进程及非法端口
1. 运行`netstat -antp`，查看服务器是否有未被授权的端口被监听，查看下对应的 pid。
 * 检查服务器是否存在恶意进程，恶意进程往往会开启监听端口，与外部控制机器进行连接。
 * 解决方法：
      1. 若发现有非授权进程，运行`ls -l /proc/$PID/exe`或`file /proc/$PID/exe `（$PID 为对应的 pid 号），查看下 pid 所对应的进程文件路径。
      2. 如果为恶意进程，删除对应的文件即可。
   * 风险性：高。
2. 使用`ps -ef`和`top`命令查看是否有异常进程
    * 检查说明：运行以上命令，当发现有名称不断变化的非授权进程占用大量系统 CPU 或内存资源时，则可能为恶意程序。
    * 解决方法：确认该进程为恶意进程后，可以使用`kill -9 进程ID`或者`pkill -9 进程名`命令结束进程，或使用防火墙限制进程外联。
    * 风险性：高。


### 三、检查恶意程序和可疑启动项
1. 使用`chkconfig --list`（Centos） 、`systemctl list-unit-files --type=service --state=enabled`（Ubuntu/Debian）命令，查看开机启动项中是否有异常的启动服务。
    * 检查说明：恶意程序往往会添加在系统的启动项，在用户关机重启后再次运行。
    * 解决方法：如发现有恶意进程，可使用` chkconfig 服务名 off` 关闭（Centos）或者使用 `systemctl disable/stop 服务名`来禁用/停止（Ubuntu/Debian），同时检查`/etc/rc.local`中是否有异常项目，如有请注释掉。
    * 风险性：高。
2. 进入 cron 文件目录，查看是否存在非法定时任务脚本。
    * 检查说明：查看`/etc/crontab`，`/etc/cron.d`，`/etc/cron.daily`，`cron.hourly/`，`cron.monthly`，`cron.weekly/`是否存在可疑脚本或程序。
    * 解决方法：如发现有不认识的计划任务，可定位脚本确认是否正常业务脚本，如果非正常业务脚本，可直接注释掉任务内容或删除脚本。
    * 风险性：高。

### 四、检查第三方软件漏洞
1. 如果您服务器内有运行 Web、数据库等应用服务，请您限制应用程序账户对文件系统的写权限，同时尽量使用非 root 账户运行。
    * 检查说明：使用非 root 账户运行，可以保障在应用程序被攻陷后，攻击者无法立即远程控制服务器，减少攻击损失。
    * 解决方法：
      1. 进入 web 服务根目录或数据库配置目录。
      2. 运行`chown -R apache:apache /var/www/xxxx`、`chmod -R 750 file1.txt`命令配置网站访问权限。
    * 风险性：中。
    * <a href=#ex>参考示例</a>
2. 升级修复应用程序漏洞
 * 检查说明：机器被入侵，部分原因是系统使用的应用程序软件版本较老，存在较多的漏洞而没有修复，导致可以被入侵利用。
 * 解决方法：比较典型的漏洞例如 ImageMagick、openssl、glibc 等，用户可以根据腾讯云已发布的安全通告指导或通过 apt-get/yum 等方式进行直接升级修复。
  * 风险性：高。

<a id="ex"></a>
**网站目录文件权限的参考示例如下：**
**场景：**
假设 HTTP 服务器运行的用户和用户组是 www，网站用户为 centos，网站根目录是`/home/centos/web`。
**方法/步骤：**
1. 我们首先设定网站目录和文件的所有者和所有组为 centos，www，如下命令：
```
 chown -R centos:www /home/centos/web
```
2. 设置网站目录权限为750，750是 centos 用户对目录拥有读写执行的权限，设置后，centos 用户可以在任何目录下创建文件，用户组有有读执行权限，这样才能进入目录，其它用户没有任何权限。
```
find -type d -exec chmod 750 {} \;
```
3. 设置网站文件权限为640，640指只有 centos 用户对网站文件有更改的权限，HTTP 服务器只有读取文件的权限，无法更改文件，其它用户无任何权限。
```
find -not -type d -exec chmod 640 {} \;
```
4. 针对个别目录设置可写权限。例如，网站的一些缓存目录就需要给 HTTP 服务有写入权限、discuz x2 的`/data/`目录就必须要写入权限。
```
find data -type d -exec chmod 770 {} \;
```

## 被入侵后的安全优化建议
- 推荐使用 SSH 密钥进行登录，减少暴力破解的风险。
- 在服务器内编辑`/etc/ssh/sshd_config`文件中的 Port 22，将 22 修改为其他非默认端口，修改之后重启 SSH 服务。可使用如下命令重启：
```
/etc/init.d/sshd restart（Centos7以下）或 systemctl restart sshd（Centos7及其以上）或 /etc/init.d/ssh restart（Debian/Ubuntu）
```
>!当修改端口时，需同时在 [云服务器控制台](https://console.cloud.tencent.com/cvm/instance/index?rid=1) 上修改对应主机安全组配置，在其入站规则中放行对应端口，详情请参见 [添加安全组规则](https://cloud.tencent.com/document/product/215/39790)。
- 如果必须使用 SSH 密码进行管理，选择一个好密码。
 * 无论应用程序管理后台（网站、中间件、tomcat 等）、远程 SSH、远程桌面、数据库，都建议设置复杂且不一样的密码。
 * 下面是一些好密码的实例（可以使用空格）：
       `1qtwo-threeMiles3c45jia`
       ` caser, lanqiu streets`
 * 下面是一些弱口令的示例，可能是您在公开的工作中常用的词或者是您生活中常用的词：
        公司名+日期（coca-cola2016xxxx）
        常用口语（Iamagoodboy）
- 使用以下命令检查主机有哪些端口开放，关闭非业务端口。
```
netstat -antp
```
- 通过**腾讯云-安全组防火墙**限制仅允许制定 IP 访问管理或通过编辑`/etc/hosts.deny`、`/etc/hosts.allow`两个文件来限制 IP。
- 应用程序尽量不使用 **root** 权限。
  例如 Apache、Redis、MySQL、Nginx 等程序，尽量不要以 root 权限的方式运行。
- 修复系统提权漏洞与运行在 root 权限下的**程序漏洞**，以免恶意软件通过漏洞提权获得 root 权限传播后门。
    * 及时更新系统或所用应用程序的版本，如 Struts2、Nginx，ImageMagick、Java 等。
    * 关闭应用程序的远程管理功能，如 Redis、NTP 等，如果无远程管理需要，可关闭对外监听端口或配置。
- 定期**备份**云服务器业务数据。
    * 对重要的业务数据进行异地备份或云备份，避免主机被入侵后无法恢复。
    * 除了您的 home，root 目录外，您还应当备份 /etc 和可用于取证的 /var/log 目录。
- 安装腾讯云**主机安全 Agent**，在发生攻击后，可以了解自身风险情况。
>?如果以上步骤均不能排查出来问题，建议您可以 [购买专家服务](https://cloud.tencent.com/document/product/296/50496) 获取专人对接服务。
