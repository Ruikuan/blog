# 加密 SQLServer 数据库备份及还原

从 SQLServer 2014 开始，就提供了原生的数据库备份加密功能。使用它不算很复杂，

## 加密备份数据库

使用下面几个步骤就可以得到一个加密的数据库备份文件。详细参考微软的[创建加密的备份](https://docs.microsoft.com/zh-cn/sql/relational-databases/backup-restore/create-an-encrypted-backup)文档

### 1. 首先创建一个 master 数据库的主密钥，数据库需要用它来加密等会儿创建的证书。

```sql
-- Creates a database master key.   
-- The key is encrypted using the password "<master key password>"  
USE master;  
GO  
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'masterKey123@';  
GO  
```

### 2. 然后创建一个备份证书，到时候备份数据库时可以选它来加密备份集。

```sql
USE master;  
GO  
CREATE CERTIFICATE MyBackupEncryptCert  
   WITH SUBJECT = '备份数据库加密证书';  
GO
```

整个证书包含公私钥。而且使用该证书加密备份的数据库必须使用该证书才能解密还原。如果只是还原到本机还好，如果需要还原数据库备份到其他机器，就需要将证书备份出来。证书备份详细可以参见[备份证书](https://docs.microsoft.com/zh-cn/sql/t-sql/statements/backup-certificate-transact-sql)文档

我们将公私钥都备份出来，备份的时候需要提供一个密码来加密私钥。

```sql
USE master;  
GO  
BACKUP CERTIFICATE MyBackupEncryptCert TO FILE = 'c:\keys\cert.cer'  
    WITH PRIVATE KEY ( FILE = 'c:\keys\cert.pvk' ,   
    ENCRYPTION BY PASSWORD = 'backupCer@Key1' );  
GO  
```

请将这个密码记下来，还原证书的时候需要提供同一个密码，否则证书还原不了，备份恢复不了就死翘翘了。

### 3. 有了证书，就可以执行加密备份了。

参考[创建加密的备份](https://docs.microsoft.com/zh-cn/sql/relational-databases/backup-restore/create-an-encrypted-backup)文档，使用下面语句就可以了。关键是使用 ENCRYPTION 选项，选择加密算法和刚才创建好的证书。

```sql
BACKUP DATABASE [MyTestDB]  
TO DISK = N'C:\DbBak\MyTestDB.bak'  
WITH  
  COMPRESSION,  
  ENCRYPTION   
   (  
    ALGORITHM = AES_256,  
    SERVER CERTIFICATE = MyBackupEncryptCert  
   ),  
  STATS = 10  
GO  
```

## 还原加密数据库备份

如果是还原到从中备份的数据库本机，是非常简单的，使用数据库还原向导，跟原来没有加密的操作一模一样。你感觉不到备份集是已经被加密过的。

但如果要还原到其他机器的数据库，在你选择备份集之后，数据库读不出来备份集里面的内容，没办法顺利还原。那么就需要下面那样做。

### 1. 首先要在目标机器数据库上还原备份证书。

将证书备份出来的文件 cert.cer 和 cert.pvk 复制到目标机器的 c:\keys\ 目录，然后执行 CREATE CERTIFICATE 语句从文件中还原证书。需要备份证书时的密码。

```sql
USE master;  
GO  
CREATE CERTIFICATE MyBackupEncryptCert   
    FROM FILE = 'c:\keys\cert.cer'   
    WITH PRIVATE KEY (FILE = 'c:\keys\cert.pvk',   
    DECRYPTION BY PASSWORD = 'backupCer@Key1');  
GO   
```

执行语句时，如果该 master 数据库先前还没有 master 数据库主密码，就需要像上面第一步那样给它创建一个。  
然后我们在目标机器数据库上有了能解密的证书。

### 2. 还原数据库

然后就很简单，执行数据库还原向导，跟没有加密一样还原就行了。


## 要注意的点

* 注意备份加密证书
* 要记住备份加密证书的密码
* 记住证书的名称