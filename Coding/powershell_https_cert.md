# 使用 PowerShell 创建自签名 https 证书

假如要创建针对域名 api.mydomain.com 的 https 证书，可以 PS 中执行如下命令即可：

```powershell
New-SelfSignedCertificate -DnsName "api.mydomain.com" -CertStoreLocation "cert:\LocalMachine\My"
```
创建的证书被存储在[证书(本地计算机)-个人]中，可以根据需要导出到其他电脑导入受信任的根证书区域，避免访问这个证书的网站不受信任。若使用 iis，则这个证书在绑定 https 协议时已经可以提供选择了。证书有效期一年。

上面命令中的 api.mydomain.com 可以更改为 *.mydomain.com 来实现通配符证书。

其他的需求如需要更改证书有效期、直接将证书创建到根证书区域等，可以参阅 [New-SelfSignedCertificate 文档](https://docs.microsoft.com/en-us/powershell/module/pkiclient/new-selfsignedcertificate?view=win10-ps)  


> [How to create a self-signed certificate for a domain name for development?](https://stackoverflow.com/a/27257921)