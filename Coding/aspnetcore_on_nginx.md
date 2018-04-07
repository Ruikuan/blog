# 在 linux 上用 nginx 代理 asp.net core 应用并使用 let's encrypt 证书开启 https

下面操作针对的是 Debian 9，基本上也可以照搬到 Ubuntu 16.04 或更新版本。我们使用的域名为 `mydomain.com`，用来接收 `let's encrypt` 证书更新提醒的邮箱是 `admin@mydomain.com`，此邮箱不需要跟使用域名一致。我们要部署的 web 应用为 `demoWeb`，为 asp.net core 2.0 应用。

## 在 linux 上安装 .net core

首先参考 https://www.microsoft.com/net/learn/get-started/linuxdebian 来将 .net core 安装到 linux 上。

安装系统组件：

```sh
sudo apt-get update
sudo apt-get install curl libunwind8 gettext apt-transport-https
```

注册可信微软产品 key：

```sh
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo mv microsoft.gpg /etc/apt/trusted.gpg.d/microsoft.gpg
```

注册微软产品 feed：

```sh
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/repos/microsoft-debian-stretch-prod stretch main" > /etc/apt/sources.list.d/dotnetdev.list'
```

安装 .NET Core SDK：

```sh
sudo apt-get update
sudo apt-get install dotnet-sdk-2.1.4
```
将 dotnet 加到环境 PATH：
```
export PATH=$PATH:$HOME/dotnet
```

创建一个 console app 并运行，验证下安装有没有问题，输出 Hello world 则表示安装成功：

```sh
dotnet new console -o myApp
cd myApp
dotnet run
```

## 将 asp.net core 应用部署到 linux

参考文档 https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?tabs=aspnetcore2x 来将应用部署到 linux。

首先 `publish` 应用 `demoWeb`，可以通过 `dotnet publish -c` 命令，或者通过 visual studio 的 publish 菜单项，将应用 publish 到一个目录中。如果是用 vs 的话，会 publish 到 `.\bin\Release\PublishOutput` 中。使用压缩软件 `winrar` 或其他将此目录中的所有文件压缩到压缩包 `demoWeb.zip`。下面我们假设这个 `demoWeb.zip` 位于 `E:\project\demoWeb\bin\Release\PublishOutput` 中。我们使用 win 10 上的 wsl 来将这个压缩文件复制到服务器。

进入 bash，在 cmd 或者 powershell 中运行：
```ps
bash
```

然后跳转到压缩包所在的目录。注意 linux 环境下，路径是区分大小写的。

```sh
cd /mnt/e/project/demoWeb/bin/Release/PublishOutput
```
**上面这个步骤可以省略，可以在压缩包所在目录按住 shift 点击右键，选择在此打开 powershell，然后在 powershell 中输入 bash，即可在 bash 中切换到当前目录。**


将压缩包复制到服务器端：
```sh
scp demoWeb.zip root@mydomain.com:/var/demoWeb
```
在服务器端解压缩：

```sh
cd /var/demoWeb
unzip demoWeb.zip
```

现在在服务器端（ssh）中输入 `dotnet /var/demoWeb/demoWeb.dll`，一切顺利的话，它会显示正在 `http://localhost:5000` listen，这个端口是在你 web 应用的 `Program.cs` 中配置的。

## 配置 nginx 反向代理

由于需要通过 nginx 进行反向代理，先修改 demoWeb 的代码，在 `Startup` 的 `Configure` 方法的 `UseAuthentication` 或类似的验证中间件前加入：

```cs
app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});
// before it
app.UseAuthentication();
```
将它重新发布。

安装 nginx：

```sh
sudo apt-get install nginx
```

启动它：
```sh
sudo service nginx start
```

可以通过 http://mydomain.com 看看能否看到 nginx 的默认页面。

配置反向代理。修改 `/etc/nginx/nginx.conf` 文件，可以用 `vim` 或者其他编辑器

```sh
sudo vim /etc/nginx/nginx.conf
```

将文件修改为：

```conf
http {
	upstream demoWeb {
		# Replace the port with your app's localhost port.
		server localhost:5000;
	}

	server {
		listen 80;
		server_name mydomain.com;

		location / {
			proxy_pass http://demoWeb;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection keep-alive;
			proxy_set_header Host $http_host;
			proxy_cache_bypass $http_upgrade;
		}
	}
}
```

保存文件，验证并重新加载 nginx 的配置：

```sh
sudo nginx -t && sudo nginx -s reload
```

如果没有问题的话，现在可以通过 http://mydomain.com 访问到你的 demoWeb 应用的内容了。

## 使用 systemd 监控 Kestrel 进程

由于 nginx 不像 iis 的 `ASP.NET Core Module` 那样能够管理 asp.net core 应用宿主 Kestrel 进程，万一这进程挂掉了，我们的网站就会 down 掉。所以我们需要使用 `systemd` 来监控它，万一它挂掉了，自动将它重新拉起来。旧版本的 linux 可以使用 `supervisor` 来做到这个。

创建服务定义文件：

```sh
sudo vim /etc/systemd/system/kestrel-demoWeb.service
```

文件内容为：

```ini
[Unit]
Description=Example .NET Web API App running on Linux

[Service]
WorkingDirectory=/var/demoWeb
ExecStart=/usr/bin/dotnet /var/demoWeb/demoWeb.dll
Restart=always
RestartSec=10  # Restart service after 10 seconds if dotnet service crashes
SyslogIdentifier=dotnet-demoWeb
User=www-data # 可以替换为其他用户名
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```

保存文件，并启用服务：

```sh
systemctl enable kestrel-demoWeb.service
```

启动服务，并验证是不是正常跑起来了：

```sh
systemctl start kestrel-demoWeb.service
systemctl status kestrel-demoWeb.service
```

顺利的话，就能跑起来了。

## 使用 let's encrypt 开启 https

此处主要参考了 https://nozzlegear.com/blog/lets-encrypt-and-nginx 来配置 https 证书。

首先安装 `Certbot` 到 `/opt/letsencrypt` 目录：

```sh
sudo apt update
sudo git clone https://github.com/certbot/certbot /opt/letsencrypt
```

创建 `/var/www/letsencrypt` 目录，并给 nginx 访问它的权限：

```sh
mkdir -p /var/www/letsencrypt
# www-data is the Nginx username
sudo chgrp www-data /var/www/letsencrypt
```

创建域名配置文件，并编辑它：

```sh
sudo mkdir -p /etc/letsencrypt/configs
sudo touch /etc/letsencrypt/configs/mydomain.com.conf
sudo vim /etc/letsencrypt/configs/mydomain.com.conf
```

内容编辑为：
```conf
# Just one single domain.
domains = mydomain.com

# Key size.
rsa-key-size = 4096 # Or 2048

# The current version of Let's Encrypt (as of May 3, 2017) is using this server.
server = https://acme-v01.api.letsencrypt.org/directory

# This email address will receive renewal reminders.
email = admin@mydomain.com

# This will run as a cronjob, so turn off the ncurses UI.
text = True

# Place the certs in the /var/www/letsencrypt folder (under .well-known/acme-challenge/).
authenticator = webroot
webroot-path = /var/www/letsencrypt/
```

修改 nginx 配置文件，让它支持 let's encrypt 的获取证书验证：

```sh
sudo vim /etc/nginx/nginx.conf
```

文件更改为：

```conf
http {
	upstream demoWeb {
		server localhost:5000;
	}

	server {
		listen 80;
		server_name mydomain.com;

		location / {
			proxy_pass http://demoWeb;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection keep-alive;
			proxy_set_header Host $http_host;
			proxy_cache_bypass $http_upgrade;
		}
		# 让 let's encrypt 的访问穿透 demoWeb 应用，让它可以获取它需要的内容
		location /.well-known/acme-challenge {
			root /var/www/letsencrypt;
		}
	}
}
```

保存文件并重新加载配置文件：

```sh
sudo nginx -t && sudo nginx -s reload
```

运行 `certbot` 来创建你的域名的证书：

```sh
cd /opt/letsencrypt
./certbot-auto --config /etc/letsencrypt/configs/mydomain.com.conf certonly
```

在处理过程中，需要你同意用户协议以及决定是否将 Email 分享，视情况自己处理。一切顺利的话，会显示一行“Congratulations! Your certificate and chain have been saved at $path”。这表明你可以在后面的 https 配置中使用这些证书文件了。

再次修改 nginx 的配置来允许 https 访问：

```sh
sudo vim /etc/nginx/nginx.conf
```

修改之后文件内容如下：

```conf
http {
	upstream demoWeb {
		server localhost:5000;
	}

	server {
		listen 80;
		server_name mydomain.com;

		location / {
			proxy_pass http://demoWeb;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection keep-alive;
			proxy_set_header Host $http_host;
			proxy_cache_bypass $http_upgrade;
		}
		# 让 let's encrypt 的访问穿透 demoWeb 应用，让它可以获取它需要的内容
		location /.well-known/acme-challenge {
			root /var/www/letsencrypt;
		}
	}
	# 加入 https 节点
	server {
		listen 443 ssl;
		server_name mydomain.com;
		ssl on;
		gzip on;
		ssl_stapling on;
		ssl_stapling_verify on;
		ssl_session_timeout 5m;
		# Note: Some tutorials may tell you to use cert.pem, which will work in most browsers but fails on e.g. Amazon Alexa server requests.
		# Alexa requests need the intermediary certs too, which cert.pem does not have. fullchain.pem does have them.
		ssl_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
		ssl_trusted_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
		ssl_certificate_key /etc/letsencrypt/live/mydomain.com/privkey.pem;

		location / {
			proxy_pass http://demoWeb;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection keep-alive;
			proxy_set_header Host $http_host;
			proxy_cache_bypass $http_upgrade;
		}
	}
}
```

保存文件并重新加载配置（再一次）：

```sh
sudo nginx -t && sudo nginx -s reload
```

顺利的话，你已经可以通过 https://mydomain.com 访问你的 demoWeb 应用了。

## 强制 https 访问 

经过上面的配置，你现在既可以通过 http 也可以通过 https 来访问自己的 web 应用。通常为了安全的考虑，会强制只通过 https 来访问，将 http 的请求跳转到 https 中。下面我们通过再一次修改 nginx 的配置来达成这一点。

```sh
sudo vim /etc/nginx/nginx.conf
```
将配置更改为：

```conf
http {
	upstream demoWeb {
		server localhost:5000;
	}

	server {
		listen 80;
		server_name mydomain.com;

		location / {
			# 将请求重定向到 https
			add_header Strict-Transport-Security max-age=15768000;
			return 301 https://$host$request_uri;
		}
		# 让 let's encrypt 的访问穿透 demoWeb 应用，让它可以获取它需要的内容
		location /.well-known/acme-challenge {
			root /var/www/letsencrypt;
		}
	}
	# 加入 https 节点
	server {
		listen 443 ssl;
		server_name mydomain.com;
		ssl on;
		gzip on;
		ssl_stapling on;
		ssl_stapling_verify on;
		ssl_session_timeout 5m;
		# Note: Some tutorials may tell you to use cert.pem, which will work in most browsers but fails on e.g. Amazon Alexa server requests.
		# Alexa requests need the intermediary certs too, which cert.pem does not have. fullchain.pem does have them.
		ssl_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
		ssl_trusted_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
		ssl_certificate_key /etc/letsencrypt/live/mydomain.com/privkey.pem;

		location / {
			proxy_pass http://demoWeb;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection keep-alive;
			proxy_set_header Host $http_host;
			proxy_cache_bypass $http_upgrade;
		}
	}
}
```

保存并重新加载配置：

```sh
sudo nginx -t && sudo nginx -s reload
```

现在访问 http://mydomain.com 的请求都会被自动重定向到 https://mydomain.com。

## 定期自动更新 let's encrypt 的证书

let's encrypt 的证书 90天 过期，因此要每隔一段时间更新一下。我们可以使用 linux 的 `cron job` 来自动完成这件事。

创建一个自动执行任务帐号有权限访问并执行的文件：

```sh
vim ~/renew-letsencrypt.sh
```

文件内容：
```sh
#!/bin/sh

mkdir -p /var/log/letsencrypt;
cd /opt/letsencrypt/;
./certbot-auto --non-interactive --keep-until-expiring --agree-tos --qui
et --config /etc/letsencrypt/configs/mydomain.com.conf certonly;

if [ $? -ne 0 ]
 then
        ERRORLOG=`tail /var/log/letsencrypt/letsencrypt.log`
        echo -e "The Let's Encrypt cert has not been renewed! \n \n" \
                 $ERRORLOG
 else
        nginx -s reload
fi

exit 0;
```

给它执行权限（不确定是不是需要）:

```sh
chmod +x ~/renew-letsencrypt.sh
```

最后，打开 `crontab` 来添加 `cron job`：

```
crontab -e -u yourUserName
# In crontab
0 0 1 JAN,MAR,MAY,JUL,SEP,NOV * ~/renew-letsencrypt.sh
```

保存即可。

至此，所有的工作都已经完成。