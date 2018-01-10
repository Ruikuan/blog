# 配置 SSR

## 初步安装配置

参见 https://github.com/koolshare/shadowsocksr/blob/manyuser/README.md

```
cp -R ~/shadowsocksr /usr/local
mkdir -p /etc/shadowsocks
```

## 设置 Supervisor

如果 down 了，自动重启。

```
apt-get install supervisor
cat >/etc/supervisor/conf.d/shadowsocksr.conf<<eof
[program:shadowsocksr]
command=/usr/bin/python /usr/local/shadowsocksr/shadowsocks/server.py -c /usr/shadowsocksr/user-config.json
autorestart=true
autostart=true
stderr_logfile=/var/log/ssrserver.err.log  
stdout_logfile=/var/log/ssrserver.out.log
user=nobody
eof
```

重启Supervisor服务

```
/etc/init.d/supervisor restart
```

查看Supervisor服务运行状态

```
supervisorctl status
```

