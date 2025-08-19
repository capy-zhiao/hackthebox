# 3\_Footprinting

## 1 前提

### 1.1 层

| **层**                    | **描述**                    | **信息类别**                                       |
| ------------------------ | ------------------------- | ---------------------------------------------- |
| `1. Internet Presence`   | 识别互联网存在和外部可访问的基础设施。       | 域名、子域名、vHost、ASN、网络块、IP 地址、云实例、安全措施            |
| `2. Gateway`             | 确定可能的安全措施来保护公司的外部和内部基础设施。 | 防火墙、DMZ、IPS/IDS、EDR、代理、NAC、网络分段、VPN、Cloudflare |
| `3. Accessible Services` | 识别可访问的外部或内部托管的接口和服务。      | 服务类型、功能、配置、端口、版本、接口                            |
| `4. Processes`           | 确定与服务相关的内部流程、来源和目的地。      | PID、已处理的数据、任务、来源、目标                            |
| `5. Privileges`          | 识别可访问服务的内部权限和特权。          | 组、用户、权限、限制、环境                                  |
| `6. OS Setup`            | 识别内部组件和系统设置。              | 操作系统类型、补丁级别、网络配置、操作系统环境、配置文件、敏感私人文件            |

### 1.2 域名

#### 1.2.1 证书透明度

```shell
curl -s https://crt.sh/\?q\=inlanefreight.com\&output\=json | jq .
```

还可以通过唯一的子域名对它们进行过滤

```shell
curl -s https://crt.sh/\?q\=inlanefreight.com\&output\=json | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

#### 1.2.2 公司托管服务器

识别那些可直接从互联网访问且非由第三方提供商托管的主机。这是因为未经第三方提供商许可，我们无法测试这些主机

```shell
for i in $(cat subdomainlist);do host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f1,4;done
```

#### 1.2.3 Shodan - IP 列表

可以通过对命令进行微调来生成一个 IP 地址列表`cut`，然后运行它们`Shodan`

```shell
for i in $(cat subdomainlist);do host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f4 >> ip-addresses.txt;done
for i in $(cat ip-addresses.txt);do shodan host $i;done
```

![image-20250707181219960](assets/image-20250707181219960.png)

我们记住了该 IP `10.129.127.22`( `matomo.inlanefreight.com`)，以便后续进行主动调查。现在，我们可以显示所有可用的 DNS 记录，以便找到更多主机。

#### 1.2.4 DNS 记录

```shell
capybaralalale@htb[/htb]$ dig any inlanefreight.com

; <<>> DiG 9.16.1-Ubuntu <<>> any inlanefreight.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52058
;; flags: qr rd ra; QUERY: 1, ANSWER: 17, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;inlanefreight.com.             IN      ANY

;; ANSWER SECTION:
inlanefreight.com.      300     IN      A       10.129.27.33
inlanefreight.com.      300     IN      A       10.129.95.250
inlanefreight.com.      3600    IN      MX      1 aspmx.l.google.com.
inlanefreight.com.      3600    IN      MX      10 aspmx2.googlemail.com.
inlanefreight.com.      3600    IN      MX      10 aspmx3.googlemail.com.
inlanefreight.com.      3600    IN      MX      5 alt1.aspmx.l.google.com.
inlanefreight.com.      3600    IN      MX      5 alt2.aspmx.l.google.com.
inlanefreight.com.      21600   IN      NS      ns.inwx.net.
inlanefreight.com.      21600   IN      NS      ns2.inwx.net.
inlanefreight.com.      21600   IN      NS      ns3.inwx.eu.
inlanefreight.com.      3600    IN      TXT     "MS=ms92346782372"
inlanefreight.com.      21600   IN      TXT     "atlassian-domain-verification=IJdXMt1rKCy68JFszSdCKVpwPN"
inlanefreight.com.      3600    IN      TXT     "google-site-verification=O7zV5-xFh_jn7JQ31"
inlanefreight.com.      300     IN      TXT     "google-site-verification=bow47-er9LdgoUeah"
inlanefreight.com.      3600    IN      TXT     "google-site-verification=gZsCG-BINLopf4hr2"
inlanefreight.com.      3600    IN      TXT     "logmein-verification-code=87123gff5a479e-61d4325gddkbvc1-b2bnfghfsed1-3c789427sdjirew63fc"
inlanefreight.com.      300     IN      TXT     "v=spf1 include:mailgun.org include:_spf.google.com include:spf.protection.outlook.com include:_spf.atlassian.net ip4:10.129.24.8 ip4:10.129.27.2 ip4:10.72.82.106 ~all"
inlanefreight.com.      21600   IN      SOA     ns.inwx.net. hostmaster.inwx.net. 2021072600 10800 3600 604800 3600

;; Query time: 332 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Mi Sep 01 18:27:22 CEST 2021
;; MSG SIZE  rcvd: 940
```

* `A`记录：我们通过 A 记录识别指向特定（子）域名的 IP 地址。这里我们只看到一个已知的 IP 地址。
* `MX`记录：邮件服务器记录显示了哪个邮件服务器负责管理公司的电子邮件。由于在本例中，这部分由 Google 负责，因此我们应该注意这一点，暂时忽略它。
* `NS`记录：这类记录显示哪些域名服务器用于将 FQDN 解析为 IP 地址。大多数托管服务提供商都使用自己的域名服务器，以便更容易识别托管服务提供商。
* `TXT`记录：此类记录通常包含不同第三方提供商的验证密钥以及 DNS 的其他安全方面，例如[SPF](https://datatracker.ietf.org/doc/html/rfc7208)、[DMARC](https://datatracker.ietf.org/doc/html/rfc7489)和[DKIM](https://datatracker.ietf.org/doc/html/rfc6376)，它们负责验证和确认所发送电子邮件的来源。如果我们仔细观察结果，就能发现一些有价值的信息。

域名信息

```shell
...SNIP... TXT     "MS=ms92346782372"
...SNIP... TXT     "atlassian-domain-verification=IJdXMt1rKCy68JFszSdCKVpwPN"
...SNIP... TXT     "google-site-verification=O7zV5-xFh_jn7JQ31"
...SNIP... TXT     "google-site-verification=bow47-er9LdgoUeah"
...SNIP... TXT     "google-site-verification=gZsCG-BINLopf4hr2"
...SNIP... TXT     "logmein-verification-code=87123gff5a479e-61d4325gddkbvc1-b2bnfghfsed1-3c789427sdjirew63fc"
...SNIP... TXT     "v=spf1 include:mailgun.org include:_spf.google.com include:spf.protection.outlook.com include:_spf.atlassian.net ip4:10.129.24.8 ip4:10.129.27.2 ip4:10.72.82.106 ~all"目前为止，我们能看到的是 DNS 服务器上的条目，乍一看并没有什么特别之处（除了一些额外的 IP 地址）。然而，我们无法看到这些条目背后的第三方提供商。我们现在能看到的核心信息是：
```

|                                         |                                           |                                             |
| --------------------------------------- | ----------------------------------------- | ------------------------------------------- |
| [Atlassian](https://www.atlassian.com/) | [谷歌 Gmail](https://www.google.com/gmail/) | [登录](https://www.logmein.com/)              |
| [\[Mailgun\]](https://www.mailgun.com/) | Outlook                                   | [INWX](https://www.inwx.com/en) ID/Username |
| 10.129.24.8                             | 10.129.27.2                               | 10.72.82.106                                |

例如，[Atlassian](https://www.atlassian.com/)表示公司使用该解决方案进行软件开发和协作。如果我们不熟悉这个平台，可以免费试用以熟悉它。

[Google Gmail](https://www.google.com/gmail/)表明 Google 用于电子邮件管理。因此，它还可以建议我们通过链接访问打开的 GDrive 文件夹或文件。

[LogMeIn](https://www.logmein.com/)是一个集中管理平台，负责在不同层面上规范和管理远程访问。然而，这种操作的集中化是一把双刃剑。如果以管理员身份获得该平台的访问权限（例如通过重复使用密码），则同时拥有对所有系统和信息的完全访问权限。

[Mailgun](https://www.mailgun.com/)提供了多种电子邮件 API、SMTP 中继和 Webhook，可用于管理电子邮件。这告诉我们要密切关注 API 接口，以便测试各种漏洞，例如 IDOR、SSRF、POST、PUT 请求以及许多其他攻击。

[Outlook](https://outlook.live.com/owa/)是文档管理的另一个指标。公司通常将 Office 365 与 OneDrive 以及 Azure Blob 和文件存储等云资源一起使用。Azure 文件存储可能非常有趣，因为它支持 SMB 协议。

我们最后看到的是[INWX](https://www.inwx.com/en)。这家公司似乎是一家提供域名购买和注册服务的主机提供商。带有“MS”值的 TXT 记录通常用于确认域名。在大多数情况下，它类似于用于登录管理平台的用户名或 ID。

### 1.3 Cloud

#### 1.3.1 公司托管服务器

```shell
capybaralalale@htb[/htb]$ for i in $(cat subdomainlist);do host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f1,4;done

blog.inlanefreight.com 10.129.24.93
inlanefreight.com 10.129.27.33
matomo.inlanefreight.com 10.129.127.22
www.inlanefreight.com 10.129.127.33
s3-website-us-west-2.amazonaws.com 10.129.95.250
```

通常，当其他员工将云存储用于管理目的时，会将其添加到 DNS 列表中。此步骤使员工能够更轻松地联系和管理它们。我们假设一家公司与我们签订了合同，并且在 IP 查找过程中，我们已经看到一个 IP 地址属于该`s3-website-us-west-2.amazonaws.com`服务器。

然而，有很多不同的方法可以找到这样的云存储。最简单、最常用的方法之一是结合使用 Google 搜索和 Google Dorks。例如，我们可以使用 Google Dorks`inurl:`将`intext:`搜索范围缩小到特定的术语。在以下示例中，我们看到包含公司名称的红色屏蔽区域。

#### 1.3.2 Google 搜索 AWS

![Google 搜索结果“intext: \[redacted\] inurl:amazonaws.com”显示了指向 Amazon S3 PDF 的链接。](assets/gsearch1.png)

#### 1.3.3 Azure 的 Google 搜索

![Google 搜索结果“intext: \[redacted\] inurl:blob.core.windows.net”显示了 Azure Blob Storage 上的 PDF 文件的链接。](assets/gsearch2.png)

这里我们已经可以看到 Google 提供的链接包含 PDF 文件。当我们搜索我们可能已经了解或想要了解的公司时，我们还会遇到其他文件，例如文本文档、演示文稿、代码等等。

此类内容通常也包含在网页源代码中，图片、JavaScript 代码或 CSS 就是从那里加载的。此过程通常可以减轻 Web 服务器的负担，避免存储不必要的内容。

#### 1.3.4 目标网站 - 源代码

![HTML 代码片段显示了具有 crossorigin 属性的 DNS 预取和预连接到 \[redacted\] blob.core.windows.net 的链接。](assets/cloud3.png)

[像domain.glass](https://domain.glass/)这样的第三方提供商也能告诉我们很多关于该公司基础设施的信息。此外，Cloudflare 的安全评估状态也被评为“安全”，这对我们来说是一个积极的信号。这意味着我们已经发现了一项值得关注的第二层（网关）安全措施。

#### 1.3.5 Domain.Glass 结果

![域名状态页面显示 Cloudflare 安全评估结果显示 \[已删除\] 安全。页面包含社交媒体链接、外部工具、IP 信息以及包含颁发机构和 DNS 名称的 SSL 证书详情。](assets/cloud1.png)

另一个非常有用的提供商是[GrayHatWarfare](https://buckets.grayhatwarfare.com/)。我们可以进行多种搜索，发现 AWS、Azure 和 GCP 云存储，甚至可以按文件格式排序和筛选。因此，一旦我们通过 Google 找到它们，我们也可以在 GrayHatWarfare 上搜索它们，并被动地发现给定云存储中存储了哪些文件。

#### 1.3.6 GrayHatWarfare 成果

![仪表板显示过滤选项和三个 AWS S3 存储桶的列表，文件数量分别为：1、73 和 0。](assets/cloud2.png)

许多公司还会使用公司名称的缩写，并在其IT基础架构中相应地使用。这些术语也是发现公司新云存储的绝佳方法之一。我们还可以同时搜索文件，查看可以同时访问的文件。

#### 1.3.7 SSH 私钥和公钥泄露

![仪表板显示 AWS S3 文件列表，其中包含两个条目：来自 \[redacted\] 存储桶的“id\_rsa”和“id\_rsa.pub”，日期为 2021 年 8 月。](assets/ghw1.png)

有时，当员工工作过度或压力过大时，一个错误就可能对整个公司造成致命影响。这些错误甚至可能导致 SSH 私钥泄露，任何人都可以下载并登录公司一台甚至多台机器，而无需使用密码。

#### 1.3.8 SSH 私钥

![RSA 私钥块的图像，以“BEGIN RSA PRIVATE KEY”开头，以“END RSA PRIVATE KEY”结尾。](assets/ghw2.png)

# 2 Host Based Enumeration

## 2.1 FTP

21: control, 20: data trans

https://en.wikipedia.org/wiki/List\_of\_FTP\_server\_return\_codes

FTP is a `clear-text` protocol that can sometimes be sniffed if conditions on the network are right.

### 2.1.1 TFTP

* `Trivial File Transfer Protocol` (`TFTP`) `does not` provide user authentication and other valuable features supported by FTP.

* FTP uses TCP, TFTP uses `UDP`

*   a few commands of `TFTP`:

    | **Commands** | **Description**                                                                                                                        |
    | ------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
    | `connect`    | Sets the remote host, and optionally the port, for file transfers.                                                                     |
    | `get`        | Transfers a file or set of files from the remote host to the local host.                                                               |
    | `put`        | Transfers a file or set of files from the local host onto the remote host.                                                             |
    | `quit`       | Exits tftp.                                                                                                                            |
    | `status`     | Shows the current status of tftp, including the current transfer mode (ascii or binary), connection status, time-out value, and so on. |
    | `verbose`    | Turns verbose mode, which displays additional information during file transfer, on or off.                                             |

    Unlike the FTP client, `TFTP` does not have directory listing functionality.

### 2.1.2 vsFTPd

```shell
sudo apt install vsftpd 
cat /etc/vsftpd.conf | grep -v "#"
```

| **Setting**                                                  | **Description**                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `listen=NO`                                                  | Run from inetd or as a standalone daemon?                    |
| `listen_ipv6=YES`                                            | Listen on IPv6 ?                                             |
| `anonymous_enable=NO`                                        | Enable Anonymous access?                                     |
| `local_enable=YES`                                           | Allow local users to login?                                  |
| `dirmessage_enable=YES`                                      | Display active directory messages when users go into certain directories? |
| `use_localtime=YES`                                          | Use local time?                                              |
| `xferlog_enable=YES`                                         | Activate logging of uploads/downloads?                       |
| `connect_from_port_20=YES`                                   | Connect from port 20?                                        |
| `secure_chroot_dir=/var/run/vsftpd/empty`                    | Name of an empty directory                                   |
| `pam_service_name=vsftpd`                                    | This string is the name of the PAM service vsftpd will use.  |
| `rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem`         | The last three options specify the location of the RSA certificate to use for SSL encrypted connections. |
| `rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key` |                                                              |
| `ssl_enable=NO`                                              |                                                              |

`/etc/ftpusers`  is used to deny certain users access to the FTP service.

<img src="assets/image-20250709173743431.png" alt="image-20250709173743431" style="width:67%;" />

One of the most common configurations of FTP servers is to allow `anonymous` access, which does not require legitimate credentials but provides access to some files. Even if we cannot download them, sometimes just listing the contents is enough to generate further ideas and note down information that will help us in another approach.

<img src="assets/image-20250709174251625.png" alt="image-20250709174251625" style="width:67%;" />

### 2.1.3 Download a File

![image-20250709174515352](assets/image-20250709174515352.png)

We also can download all the files and folders we have access to at once. This is especially useful if the FTP server has many different files in a larger folder structure. However, this can cause alarms because no one from the company usually wants to download all files and content all at once.

```shell
wget -m --no-passive ftp://anonymous:anonymous@10.129.14.136
```

### 2.1.4 Upload a File

![image-20250709174716716](assets/image-20250709174716716.png)

### 2.1.5 Footprinting the Service

 More information on the capabilities of Nmap and NSE can be found in the [Network Enumeration with Nmap](https://academy.hackthebox.com/course/preview/network-enumeration-with-nmap) module. We can update this database of NSE scripts with the command shown.

```shell
sudo nmap --script-updatedb
```

All the NSE scripts are located on the Pwnbox in `/usr/share/nmap/scripts/`, but on our systems, we can find them using a simple command.

```shell
find / -type f -name ftp* 2>/dev/null | grep scripts
```

<img src="assets/image-20250709174852803.png" alt="image-20250709174852803" style="width:67%;" />

As we already know, the FTP server usually runs on the standard TCP port 21, which we can scan using Nmap. We also use the version scan (`-sV`), aggressive scan (`-A`), and the default script scan (`-sC`) against our target `10.129.14.136`.

![image-20250709175217255](assets/image-20250709175217255.png)

Nmap also provides the ability to trace the progress of NSE scripts at the network level if we use the `--script-trace` option in our scans. This lets us see what commands Nmap sends, what ports are used, and what responses we receive from the scanned server.

### 2.1.6 Service Interaction

```shell
capybaralalale@htb[/htb]$ nc -nv 10.129.14.136 21
```

这是用 **netcat (nc)** 工具连接到目标主机的 **21 端口**（FTP 默认端口）。
 参数解释：

- `-n`：不解析 DNS（避免延迟）
- `-v`：显示连接的详细信息（verbose）

#### ✅ 用途：

- 测试 FTP 端口是否开放
- 快速判断服务是否存在
- 有时可以手动输入 FTP 命令测试交互（像模拟 telnet 一样）

```shell
capybaralalale@htb[/htb]$ telnet 10.129.14.136 21
```

- 手动交互（输入 FTP 命令，如 `USER anonymous`、`PASS test`）
- 查看服务 banner，比如：

```
220 (vsFTPd 3.0.3)
```

It looks slightly different if the FTP server runs with TLS/SSL encryption. Because then we need a client that can handle TLS/SSL. For this, we can use the client `openssl` and communicate with the FTP server. The good thing about using `openssl` is that we can see the SSL certificate, which can also be helpful.

```shell
capybaralalale@htb[/htb]$ openssl s_client -connect 10.129.14.136:21 -starttls ftp

CONNECTED(00000003)                                                                                      
Can't use SSL_get_servername                        
depth=0 C = US, ST = California, L = Sacramento, O = Inlanefreight, OU = Dev, CN = master.inlanefreight.htb, emailAddress = admin@inlanefreight.htb
verify error:num=18:self signed certificate
verify return:1

depth=0 C = US, ST = California, L = Sacramento, O = Inlanefreight, OU = Dev, CN = master.inlanefreight.htb, emailAddress = admin@inlanefreight.htb
verify return:1
---                                                 
Certificate chain
 0 s:C = US, ST = California, L = Sacramento, O = Inlanefreight, OU = Dev, CN = master.inlanefreight.htb, emailAddress = admin@inlanefreight.htb
 
 i:C = US, ST = California, L = Sacramento, O = Inlanefreight, OU = Dev, CN = master.inlanefreight.htb, emailAddress = admin@inlanefreight.htb
---
 
Server certificate

-----BEGIN CERTIFICATE-----

MIIENTCCAx2gAwIBAgIUD+SlFZAWzX5yLs2q3ZcfdsRQqMYwDQYJKoZIhvcNAQEL
...SNIP...
```

## 2.2 SMB

`Server Message Block` (`SMB`) is a client-server protocol that regulates access to files and entire directories and other network resources such as printers, routers, or interfaces released for the network.

uses TCP

### 2.2.1 Samba

As mentioned earlier, there is an alternative implementation of the SMB server called Samba, which is developed for Unix-based operating systems. Samba implements the Common Internet File System (`CIFS`) network protocol. [CIFS](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-cifs/934c2faa-54af-4526-ac74-6a24d126724e) is a dialect of SMB, meaning it is a specific implementation of the SMB protocol originally created by Microsoft. This allows Samba to communicate effectively with newer Windows systems. Therefore, it is often referred to as SMB/CIFS.

#### 2.2.1.1 Default Configuration

```shell
cat /etc/samba/smb.conf | grep -v "#\|\;" 
```

| **Setting**                    | **Description**                                              |
| ------------------------------ | ------------------------------------------------------------ |
| `[sharename]`                  | The name of the network share.                               |
| `workgroup = WORKGROUP/DOMAIN` | Workgroup that will appear when clients query.               |
| `path = /path/here/`           | The directory to which user is to be given access.           |
| `server string = STRING`       | The string that will show up when a connection is initiated. |
| `unix password sync = yes`     | Synchronize the UNIX password with the SMB password?         |
| `usershare allow guests = yes` | Allow non-authenticated users to access defined share?       |
| `map to guest = bad user`      | What to do when a user login request doesn't match a valid UNIX user? |
| `browseable = yes`             | Should this share be shown in the list of available shares?  |
| `guest ok = yes`               | Allow connecting to the service without using a password?    |
| `read only = yes`              | Allow users to read files only?                              |
| `create mask = 0700`           | What permissions need to be set for newly created files?     |

### 2.2.2 SMBclient

Now we can display a list (`-L`) of the server's shares with the `smbclient` command from our host. We use the so-called `null session` (`-N`), which is `anonymous` access without the input of existing users or valid passwords.

```shell
smbclient -N -L //10.129.55.45
smbclient //10.129.14.128/notes
```

Once we have discovered interesting files or folders, we can download them using the `get` command. Smbclient also allows us to execute local system commands using an exclamation mark at the beginning (`!<cmd>`) without interrupting the connection.

![image-20250709183416916](assets/image-20250709183416916.png)

From the administrative point of view, we can check these connections using `smbstatus`. Apart from the Samba version, we can also see who, from which host, and which share the client is connected. This is especially important once we have entered a subnet (perhaps even an isolated one) that the others can still access.



### 2.2.3 nmap

```shell
sudo nmap 10.129.14.128 -sV -sC -p139,445
```

![image-20250709183201088](assets/image-20250709183201088.png)

### 2.2.4 rpc

The [Remote Procedure Call](https://www.geeksforgeeks.org/remote-procedure-call-rpc-in-operating-system/) (`RPC`) is a concept and, therefore, also a central tool to realize operational and work-sharing structures in networks and client-server architectures. The communication process via RPC includes passing parameters and the return of a function value.

```shell
capybaralalale@htb[/htb]$ rpcclient -U "" 10.129.14.128

Enter WORKGROUP\'s password:
rpcclient $> 
```

The `rpcclient` offers us many different requests with which we can execute specific functions on the SMB server to get information. A complete list of all these functions can be found on the [man page](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) of the rpcclient.

| **Query**                 | **Description**                                              |
| ------------------------- | ------------------------------------------------------------ |
| `srvinfo`                 | Server information.                                          |
| `enumdomains`             | Enumerate all domains that are deployed in the network.      |
| `querydominfo`            | Provides domain, server, and user information of deployed domains. |
| `netshareenumall`         | Enumerates all available shares.                             |
| `netsharegetinfo <share>` | Provides information about a specific share.                 |
| `enumdomusers`            | Enumerates all domain users.                                 |
| `queryuser <RID>`         | Provides information about a specific user.                  |

即使远程 Windows 主机限制了很多 `rpcclient` 命令，我们依然可以通过 **暴力枚举 RID（用户ID）** 的方式，利用 `queryuser` 命令来发现系统中有哪些用户存在。

```shell
for i in $(seq 500 1100);do rpcclient -N -U "" 10.129.14.128 -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";done
        User Name   :   sambauser
        user_rid :      0x1f5
        group_rid:      0x201
		
        User Name   :   mrb3n
        user_rid :      0x3e8
        group_rid:      0x201
		
        User Name   :   cry0l1t3
        user_rid :      0x3e9
        group_rid:      0x201
```

An alternative to this would be a Python script from [Impacket](https://github.com/SecureAuthCorp/impacket) called [samrdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/samrdump.py).

```shell
samrdump.py 10.129.14.128
```

The information we have already obtained with `rpcclient` can also be obtained using other tools. For example, the [SMBMap](https://github.com/ShawnDEvans/smbmap) and [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) tools are also widely used and helpful for the enumeration of SMB services.

```shell
smbmap -H 10.129.14.128
[+] Finding open SMB ports....
[+] User SMB session established on 10.129.14.128...
[+] IP: 10.129.14.128:445       Name: 10.129.14.128                                     
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        home                                                    NO ACCESS       INFREIGHT Samba
        dev                                                     NO ACCESS       DEVenv
        notes                                                   NO ACCESS       CheckIT
        IPC$                                                    NO ACCESS       IPC Service (DEVSM)
```



```shell
crackmapexec smb 10.129.14.128 --shares -u '' -p ''
SMB         10.129.14.128   445    DEVSMB           [*] Windows 6.1 Build 0 (name:DEVSMB) (domain:) (signing:False) (SMBv1:False)
SMB         10.129.14.128   445    DEVSMB           [+] \: 
SMB         10.129.14.128   445    DEVSMB           [+] Enumerated shares
SMB         10.129.14.128   445    DEVSMB           Share           Permissions     Remark
SMB         10.129.14.128   445    DEVSMB           -----           -----------     ------
SMB         10.129.14.128   445    DEVSMB           print$                          Printer Drivers
SMB         10.129.14.128   445    DEVSMB           home                            INFREIGHT Samba
SMB         10.129.14.128   445    DEVSMB           dev                             DEVenv
SMB         10.129.14.128   445    DEVSMB           notes           READ,WRITE      CheckIT
SMB         10.129.14.128   445    DEVSMB           IPC$                            IPC Service (DEVSM)
```

Another tool worth mentioning is the so-called [enum4linux-ng](https://github.com/cddmp/enum4linux-ng), which is based on an older tool, enum4linux. This tool automates many of the queries, but not all, and can return a large amount of information.

```shell
capybaralalale@htb[/htb]$ git clone https://github.com/cddmp/enum4linux-ng.git
capybaralalale@htb[/htb]$ cd enum4linux-ng
capybaralalale@htb[/htb]$ pip3 install -r requirements.txt
./enum4linux-ng.py 10.129.14.128 -A
```

## 2.3 NFS

`Network File System` (`NFS`) is a network file system developed by Sun Microsystems and has the same purpose as SMB.

NFSV4比其前代产品的重要优点是，只有一个UDP或TCP端口2049用于运行该服务

NFS is based on the [Open Network Computing Remote Procedure Call](https://en.wikipedia.org/wiki/Sun_RPC) (`ONC-RPC`/`SUN-RPC`) protocol exposed on `TCP` and `UDP` ports `111`, which uses [External Data Representation](https://en.wikipedia.org/wiki/External_Data_Representation) (`XDR`) for the system-independent exchange of data. 

### 2.3.1 Dangerous Settings

However, even with NFS, some settings can be dangerous for the company and its infrastructure. Here are some of them listed:

| **Option**       | **Description**                                              |
| ---------------- | ------------------------------------------------------------ |
| `rw`             | Read and write permissions.                                  |
| `insecure`       | Ports above 1024 will be used.                               |
| `nohide`         | If another file system was mounted below an exported directory, this directory is exported by its own exports entry. |
| `no_root_squash` | All files created by root are kept with the UID/GID 0.       |

the `insecure` option. This is dangerous because users can use ports above 1024. The first 1024 ports can only be used by root. This prevents the fact that no users can use sockets above port 1024 for the NFS service and interact with it.

### 2.3.2 Footprinting the Service

When footprinting NFS, the TCP ports `111` and `2049` are essential. We can also get information about the NFS service and the host via RPC, as shown below in the example.

<img src="assets/image-20250711211314623.png" alt="image-20250711211314623" style="width:67%;" />

The `rpcinfo` NSE script retrieves a list of all currently running RPC services, their names and descriptions, and the ports they use. This lets us check whether the target share is connected to the network on all required ports. Also, for NFS, Nmap has some NSE scripts that can be used for the scans. These can then show us, for example, the `contents` of the share and its `stats`.

![image-20250711211423425](assets/image-20250711211423425.png)

### 2.3.3 NFS Shares

Once we have discovered such an NFS service, we can mount it on our local machine. For this, we can create a new empty folder to which the NFS share will be mounted. Once mounted, we can navigate it and view the contents just like our local system.

![image-20250711211605691](assets/image-20250711211605691.png)

![image-20250711211734059](assets/image-20250711211734059.png)

```shell
sudo umount ./target-NFS
```

## 2.4 dns

There are several types of DNS servers that are used worldwide:

- DNS root server
- Authoritative name server
- Non-authoritative name server
- Caching server
- Forwarding server
- Resolver

| **Server Type**                | **Description**                                              |
| ------------------------------ | ------------------------------------------------------------ |
| `DNS Root Server`              | The root servers of the DNS are responsible for the top-level domains (`TLD`). As the last instance, they are only requested if the name server does not respond. Thus, a root server is a central interface between users and content on the Internet, as it links domain and IP address. The [Internet Corporation for Assigned Names and Numbers](https://www.icann.org/) (`ICANN`) coordinates the work of the root name servers. There are `13` such root servers around the globe. |
| `Authoritative Nameserver`     | Authoritative name servers hold authority for a particular zone. They only answer queries from their area of responsibility, and their information is binding. If an authoritative name server cannot answer a client's query, the root name server takes over at that point. Based on the country, company, etc., authoritative nameservers provide answers to recursive DNS nameservers, assisting in finding the specific web server(s). |
| `Non-authoritative Nameserver` | Non-authoritative name servers are not responsible for a particular DNS zone. Instead, they collect information on specific DNS zones themselves, which is done using recursive or iterative DNS querying. |
| `Caching DNS Server`           | Caching DNS servers cache information from other name servers for a specified period. The authoritative name server determines the duration of this storage. |
| `Forwarding Server`            | Forwarding servers perform only one function: they forward DNS queries to another DNS server. |
| `Resolver`                     | Resolvers are not authoritative DNS servers but perform name resolution locally in the computer or router. |

DNS is mainly unencrypted. Devices on the local WLAN and Internet providers can therefore hack in and spy on DNS queries. Since this poses a privacy risk, there are now some solutions for DNS encryption. By default, IT security professionals apply `DNS over TLS` (`DoT`) or `DNS over HTTPS` (`DoH`) here. In addition, the network protocol `DNSCrypt` also encrypts the traffic between the computer and the name server.

However, the DNS does not only link computer names and IP addresses. It also stores and outputs additional information about the services associated with a domain. A DNS query can therefore also be used, for example, to determine which computer serves as the e-mail server for the domain in question or what the domain's name servers are called.

![Diagram showing domain hierarchy: Root, Top Level Domains (TLD) like net, org, com, dev, io; Second Level Domain inlanefreight.com; Sub-Domains dev.inlanefreight.com, www.inlanefreight.com, mail.inlanefreight.com; Host WS01.dev.inlanefreight.com.](assets/tooldev-dns.png)

Different `DNS records` are used for the DNS queries, which all have various tasks. Moreover, separate entries exist for different functions since we can set up mail servers and other servers for a domain.

| **DNS Record** | **Description**                                              |
| -------------- | ------------------------------------------------------------ |
| `A`            | Returns an IPv4 address of the requested domain as a result. |
| `AAAA`         | Returns an IPv6 address of the requested domain.             |
| `MX`           | Returns the responsible mail servers as a result.            |
| `NS`           | Returns the DNS servers (nameservers) of the domain.         |
| `TXT`          | This record can contain various information. The all-rounder can be used, e.g., to validate the Google Search Console or validate SSL certificates. In addition, SPF and DMARC entries are set to validate mail traffic and protect it from spam. |
| `CNAME`        | This record serves as an alias for another domain name. If you want the domain www.hackthebox.eu to point to the same IP as hackthebox.eu, you would create an A record for hackthebox.eu and a CNAME record for www.hackthebox.eu. |
| `PTR`          | The PTR record works the other way around (reverse lookup). It converts IP addresses into valid domain names. |
| `SOA`          | Provides information about the corresponding DNS zone and email address of the administrative contact. |

The `SOA` record is located in a domain's zone file and specifies who is responsible for the operation of the domain and how DNS information for the domain is managed.

```shell
capybaralalale@htb[/htb]$ dig soa www.inlanefreight.com

; <<>> DiG 9.16.27-Debian <<>> soa www.inlanefreight.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15876
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;www.inlanefreight.com.         IN      SOA

;; AUTHORITY SECTION:
inlanefreight.com.      900     IN      SOA     ns-161.awsdns-20.com. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400

;; Query time: 16 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Thu Jan 05 12:56:10 GMT 2023
;; MSG SIZE  rcvd: 128
```

The dot (.) is replaced by an at sign (@) in the email address. In this example, the email address of the administrator is `awsdns-hostmaster@amazon.com`.

#### Default Configuration

All DNS servers work with three different types of configuration files:

1. local DNS configuration files
2. zone files
3. reverse name resolution files

The DNS server [Bind9](https://www.isc.org/bind/) is very often used on Linux-based distributions. Its local configuration file (`named.conf`) is roughly divided into two sections, firstly the options section for general settings and secondly the zone entries for the individual domains. The local configuration files are usually:

- `named.conf.local`
- `named.conf.options`
- `named.conf.log`

It contains the associated RFC where we can customize the server to our needs and our domain structure with the individual zones for different domains. The configuration file `named.conf` is divided into several options that control the behavior of the name server. A distinction is made between `global options` and `zone options`.

Global options are general and affect all zones. A zone option only affects the zone to which it is assigned. Options not listed in named.conf have default values. If an option is both global and zone-specific, then the zone option takes precedence.

#### Local DNS Configuration

```shell
root@bind9:~# cat /etc/bind/named.conf.local

//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone "domain.com" {
    type master;
    file "/etc/bind/db.domain.com";
    allow-update { key rndc-key; };
};
```

In this file, we can define the different zones. These zones are divided into individual files, which in most cases are mainly intended for one domain only. Exceptions are ISP and public DNS servers. In addition, many different options extend or reduce the functionality. We can look these up on the [documentation](https://wiki.debian.org/Bind9) of Bind9.

A `zone file` is a text file that describes a DNS zone with the BIND file format. In other words it is a point of delegation in the DNS tree. The BIND file format is the industry-preferred zone file format and is now well established in DNS server software. A zone file describes a zone completely. There must be precisely one `SOA` record and at least one `NS` record. The SOA resource record is usually located at the beginning of a zone file. The main goal of these global rules is to improve the readability of zone files. A syntax error usually results in the entire zone file being considered unusable. The name server behaves similarly as if this zone did not exist. It responds to DNS queries with a `SERVFAIL` error message.

In short, here, all `forward records` are entered according to the BIND format. This allows the DNS server to identify which domain, hostname, and role the IP addresses belong to. In simple terms, this is the phone book where the DNS server looks up the addresses for the domains it is searching for.

#### Zone Files

```shell
root@bind9:~# cat /etc/bind/db.domain.com

;
; BIND reverse data file for local loopback interface
;
$ORIGIN domain.com
$TTL 86400
@     IN     SOA    dns1.domain.com.     hostmaster.domain.com. (
                    2001062501 ; serial
                    21600      ; refresh after 6 hours
                    3600       ; retry after 1 hour
                    604800     ; expire after 1 week
                    86400 )    ; minimum TTL of 1 day

      IN     NS     ns1.domain.com.
      IN     NS     ns2.domain.com.

      IN     MX     10     mx.domain.com.
      IN     MX     20     mx2.domain.com.

             IN     A       10.129.14.5

server1      IN     A       10.129.14.5
server2      IN     A       10.129.14.7
ns1          IN     A       10.129.14.2
ns2          IN     A       10.129.14.3

ftp          IN     CNAME   server1
mx           IN     CNAME   server1
mx2          IN     CNAME   server2
www          IN     CNAME   server2
```

For the IP address to be resolved from the `Fully Qualified Domain Name` (`FQDN`), the DNS server must have a reverse lookup file. In this file, the computer name (FQDN) is assigned to the last octet of an IP address, which corresponds to the respective host, using a `PTR` record. The PTR records are responsible for the reverse translation of IP addresses into names, as we have already seen in the above table.

#### Reverse Name Resolution Zone Files

```shell
root@bind9:~# cat /etc/bind/db.10.129.14

;
; BIND reverse data file for local loopback interface
;
$ORIGIN 14.129.10.in-addr.arpa
$TTL 86400
@     IN     SOA    dns1.domain.com.     hostmaster.domain.com. (
                    2001062501 ; serial
                    21600      ; refresh after 6 hours
                    3600       ; retry after 1 hour
                    604800     ; expire after 1 week
                    86400 )    ; minimum TTL of 1 day

      IN     NS     ns1.domain.com.
      IN     NS     ns2.domain.com.

5    IN     PTR    server1.domain.com.
7    IN     MX     mx.domain.com.
...SNIP...
```

#### Dangerous Settings

There are many ways in which a DNS server can be attacked. For example, a list of vulnerabilities targeting the BIND9 server can be found at [CVEdetails](https://www.cvedetails.com/product/144/ISC-Bind.html?vendor_id=64). In addition, SecurityTrails provides a short [list](https://web.archive.org/web/20250329174745/https://securitytrails.com/blog/most-popular-types-dns-attacks) of the most popular attacks on DNS servers.

| **Option**        | **Description**                                              |
| ----------------- | ------------------------------------------------------------ |
| `allow-query`     | Defines which hosts are allowed to send requests to the DNS server. |
| `allow-recursion` | Defines which hosts are allowed to send recursive requests to the DNS server. |
| `allow-transfer`  | Defines which hosts are allowed to receive zone transfers from the DNS server. |
| `zone-statistics` | Collects statistical data of zones.                          |

#### Footprinting the Service

The footprinting at DNS servers is done as a result of the requests we send. So, first of all, the DNS server can be queried as to which other name servers are known. We do this using the NS record and the specification of the DNS server we want to query using the `@` character. This is because if there are other DNS servers, we can also use them and query the records. However, other DNS servers may be configured differently and, in addition, may be permanent for other zones.

#### DIG - NS Query

```shell
capybaralalale@htb[/htb]$ dig ns inlanefreight.htb @10.129.14.128

; <<>> DiG 9.16.1-Ubuntu <<>> ns inlanefreight.htb @10.129.14.128
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45010
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: ce4d8681b32abaea0100000061475f73842c401c391690c7 (good)
;; QUESTION SECTION:
;inlanefreight.htb.             IN      NS

;; ANSWER SECTION:
inlanefreight.htb.      604800  IN      NS      ns.inlanefreight.htb.

;; ADDITIONAL SECTION:
ns.inlanefreight.htb.   604800  IN      A       10.129.34.136

;; Query time: 0 msec
;; SERVER: 10.129.14.128#53(10.129.14.128)
;; WHEN: So Sep 19 18:04:03 CEST 2021
;; MSG SIZE  rcvd: 107
```

Sometimes it is also possible to query a DNS server's version using a class CHAOS query and type TXT. However, this entry must exist on the DNS server. For this, we could use the following command:

#### DIG - Version Query

```shell
capybaralalale@htb[/htb]$ dig CH TXT version.bind 10.129.120.85

; <<>> DiG 9.10.6 <<>> CH TXT version.bind
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47786
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
version.bind.       0       CH      TXT     "9.10.6-P1"

;; ADDITIONAL SECTION:
version.bind.       0       CH      TXT     "9.10.6-P1-Debian"

;; Query time: 2 msec
;; SERVER: 10.129.120.85#53(10.129.120.85)
;; WHEN: Wed Jan 05 20:23:14 UTC 2023
;; MSG SIZE  rcvd: 101
```

We can use the option `ANY` to view all available records. This will cause the server to show us all available entries that it is willing to disclose. It is important to note that not all entries from the zones will be shown.

#### DIG - ANY Query

```shell
capybaralalale@htb[/htb]$ dig any inlanefreight.htb @10.129.14.128

; <<>> DiG 9.16.1-Ubuntu <<>> any inlanefreight.htb @10.129.14.128
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7649
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 5, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 064b7e1f091b95120100000061476865a6026d01f87d10ca (good)
;; QUESTION SECTION:
;inlanefreight.htb.             IN      ANY

;; ANSWER SECTION:
inlanefreight.htb.      604800  IN      TXT     "v=spf1 include:mailgun.org include:_spf.google.com include:spf.protection.outlook.com include:_spf.atlassian.net ip4:10.129.124.8 ip4:10.129.127.2 ip4:10.129.42.106 ~all"
inlanefreight.htb.      604800  IN      TXT     "atlassian-domain-verification=t1rKCy68JFszSdCKVpw64A1QksWdXuYFUeSXKU"
inlanefreight.htb.      604800  IN      TXT     "MS=ms97310371"
inlanefreight.htb.      604800  IN      SOA     inlanefreight.htb. root.inlanefreight.htb. 2 604800 86400 2419200 604800
inlanefreight.htb.      604800  IN      NS      ns.inlanefreight.htb.

;; ADDITIONAL SECTION:
ns.inlanefreight.htb.   604800  IN      A       10.129.34.136

;; Query time: 0 msec
;; SERVER: 10.129.14.128#53(10.129.14.128)
;; WHEN: So Sep 19 18:42:13 CEST 2021
;; MSG SIZE  rcvd: 437
```

`Zone transfer` refers to the transfer of zones to another server in DNS, which generally happens over TCP port 53. This procedure is abbreviated `Asynchronous Full Transfer Zone` (`AXFR`). Since a DNS failure usually has severe consequences for a company, the zone file is almost invariably kept identical on several name servers. When changes are made, it must be ensured that all servers have the same data. Synchronization between the servers involved is realized by zone transfer. Using a secret key `rndc-key`, which we have seen initially in the default configuration, the servers make sure that they communicate with their own master or slave. Zone transfer involves the mere transfer of files or records and the detection of discrepancies in the data sets of the servers involved.

The original data of a zone is located on a DNS server, which is called the `primary` name server for this zone. However, to increase the reliability, realize a simple load distribution, or protect the primary from attacks, one or more additional servers are installed in practice in almost all cases, which are called `secondary` name servers for this zone. For some `Top-Level Domains` (`TLDs`), making zone files for the `Second Level Domains` accessible on at least two servers is mandatory.

DNS entries are generally only created, modified, or deleted on the primary. This can be done by manually editing the relevant zone file or automatically by a dynamic update from a database. A DNS server that serves as a direct source for synchronizing a zone file is called a master. A DNS server that obtains zone data from a master is called a slave. A primary is always a master, while a secondary can be both a slave and a master.

The slave fetches the `SOA` record of the relevant zone from the master at certain intervals, the so-called refresh time, usually one hour, and compares the serial numbers. If the serial number of the SOA record of the master is greater than that of the slave, the data sets no longer match.

> **Zone transfer（AXFR）** 是 DNS 系统中主从服务器用来同步域名记录的一种机制，通过 TCP 传输整个区域的数据，确保多个 DNS 服务器数据一致。

#### DIG - AXFR Zone Transfer

```shell
capybaralalale@htb[/htb]$ dig axfr inlanefreight.htb @10.129.14.128

; <<>> DiG 9.16.1-Ubuntu <<>> axfr inlanefreight.htb @10.129.14.128
;; global options: +cmd
inlanefreight.htb.      604800  IN      SOA     inlanefreight.htb. root.inlanefreight.htb. 2 604800 86400 2419200 604800
inlanefreight.htb.      604800  IN      TXT     "MS=ms97310371"
inlanefreight.htb.      604800  IN      TXT     "atlassian-domain-verification=t1rKCy68JFszSdCKVpw64A1QksWdXuYFUeSXKU"
inlanefreight.htb.      604800  IN      TXT     "v=spf1 include:mailgun.org include:_spf.google.com include:spf.protection.outlook.com include:_spf.atlassian.net ip4:10.129.124.8 ip4:10.129.127.2 ip4:10.129.42.106 ~all"
inlanefreight.htb.      604800  IN      NS      ns.inlanefreight.htb.
app.inlanefreight.htb.  604800  IN      A       10.129.18.15
internal.inlanefreight.htb. 604800 IN   A       10.129.1.6
mail1.inlanefreight.htb. 604800 IN      A       10.129.18.201
ns.inlanefreight.htb.   604800  IN      A       10.129.34.136
inlanefreight.htb.      604800  IN      SOA     inlanefreight.htb. root.inlanefreight.htb. 2 604800 86400 2419200 604800
;; Query time: 4 msec
;; SERVER: 10.129.14.128#53(10.129.14.128)
;; WHEN: So Sep 19 18:51:19 CEST 2021
;; XFR size: 9 records (messages 1, bytes 520)
```

If the administrator used a subnet for the `allow-transfer` option for testing purposes or as a workaround solution or set it to `any`, everyone would query the entire zone file at the DNS server. In addition, other zones can be queried, which may even show internal IP addresses and hostnames.

#### DIG - AXFR Zone Transfer - Internal

```shell
capybaralalale@htb[/htb]$ dig axfr internal.inlanefreight.htb @10.129.14.128

; <<>> DiG 9.16.1-Ubuntu <<>> axfr internal.inlanefreight.htb @10.129.14.128
;; global options: +cmd
internal.inlanefreight.htb. 604800 IN   SOA     inlanefreight.htb. root.inlanefreight.htb. 2 604800 86400 2419200 604800
internal.inlanefreight.htb. 604800 IN   TXT     "MS=ms97310371"
internal.inlanefreight.htb. 604800 IN   TXT     "atlassian-domain-verification=t1rKCy68JFszSdCKVpw64A1QksWdXuYFUeSXKU"
internal.inlanefreight.htb. 604800 IN   TXT     "v=spf1 include:mailgun.org include:_spf.google.com include:spf.protection.outlook.com include:_spf.atlassian.net ip4:10.129.124.8 ip4:10.129.127.2 ip4:10.129.42.106 ~all"
internal.inlanefreight.htb. 604800 IN   NS      ns.inlanefreight.htb.
dc1.internal.inlanefreight.htb. 604800 IN A     10.129.34.16
dc2.internal.inlanefreight.htb. 604800 IN A     10.129.34.11
mail1.internal.inlanefreight.htb. 604800 IN A   10.129.18.200
ns.internal.inlanefreight.htb. 604800 IN A      10.129.34.136
vpn.internal.inlanefreight.htb. 604800 IN A     10.129.1.6
ws1.internal.inlanefreight.htb. 604800 IN A     10.129.1.34
ws2.internal.inlanefreight.htb. 604800 IN A     10.129.1.35
wsus.internal.inlanefreight.htb. 604800 IN A    10.129.18.2
internal.inlanefreight.htb. 604800 IN   SOA     inlanefreight.htb. root.inlanefreight.htb. 2 604800 86400 2419200 604800
;; Query time: 0 msec
;; SERVER: 10.129.14.128#53(10.129.14.128)
;; WHEN: So Sep 19 18:53:11 CEST 2021
;; XFR size: 15 records (messages 1, bytes 664)
```

The individual `A` records with the hostnames can also be found out with the help of a brute-force attack. To do this, we need a list of possible hostnames, which we use to send the requests in order. Such lists are provided, for example, by [SecLists](https://github.com/danielmiessler/SecLists/blob/master/Discovery/DNS/subdomains-top1million-5000.txt).

An option would be to execute a `for-loop` in Bash that lists these entries and sends the corresponding query to the desired DNS server.

#### Subdomain Brute Forcing

```shell
capybaralalale@htb[/htb]$ for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt);do dig $sub.inlanefreight.htb @10.129.14.128 | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt;done

ns.inlanefreight.htb.   604800  IN      A       10.129.34.136
mail1.inlanefreight.htb. 604800 IN      A       10.129.18.201
app.inlanefreight.htb.  604800  IN      A       10.129.18.15
```

Many different tools can be used for this, and most of them work in the same way. One of these tools is, for example [DNSenum](https://github.com/fwaeytens/dnsenum).

```shell
capybaralalale@htb[/htb]$ dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt inlanefreight.htb

dnsenum VERSION:1.2.6

-----   inlanefreight.htb   -----


Host's addresses:
__________________



Name Servers:
______________

ns.inlanefreight.htb.                    604800   IN    A        10.129.34.136


Mail (MX) Servers:
___________________



Trying Zone Transfers and getting Bind Versions:
_________________________________________________

unresolvable name: ns.inlanefreight.htb at /usr/bin/dnsenum line 900 thread 1.

Trying Zone Transfer for inlanefreight.htb on ns.inlanefreight.htb ...
AXFR record query failed: no nameservers


Brute forcing with /home/cry0l1t3/Pentesting/SecLists/Discovery/DNS/subdomains-top1million-110000.txt:
_______________________________________________________________________________________________________

ns.inlanefreight.htb.                    604800   IN    A        10.129.34.136
mail1.inlanefreight.htb.                 604800   IN    A        10.129.18.201
app.inlanefreight.htb.                   604800   IN    A        10.129.18.15
ns.inlanefreight.htb.                    604800   IN    A        10.129.34.136

...SNIP...
done.
```

#### q s

Interact with the target DNS using its IP address and enumerate the FQDN of it for the "inlanefreight.htb" domain.

dig @10.129.47.225 inlanefreight.htb AXFR

identify if its possible to perform a zone transfer and submit the TXT record as the answer. (Format: HTB{...})

└──╼ [★]$ dig @10.129.47.225 internal.inlanefreight.htb AXFR

What is the IPv4 address of the hostname DC1?



What is the FQDN of the host where the last octet ends with "x.x.x.203"?

dnsenum --dnsserver 10.129.47.225 --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt inlanefreight.htb

dnsenum --dnsserver 10.129.47.225 --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/SecLists/Discovery/DNS/fierce-hostlist.txt dev.inlanefreight.htb

## 2.5 smtp

The `Simple Mail Transfer Protocol` (`SMTP`) is a protocol for sending emails in an IP network. SMTP is often combined with the IMAP or POP3 protocols]

port `25`. newer SMTP servers also use other ports such as TCP port `587`. 

On arrival at the destination SMTP server, the data packets are reassembled to form a complete e-mail. From there, the `Mail delivery agent` (`MDA`) transfers it to the recipient's mailbox.

| Client (`MUA`) | `➞`  | Submission Agent (`MSA`) | `➞`  | Open Relay (`MTA`) | `➞`  | Mail Delivery Agent (`MDA`) | `➞`  | Mailbox (`POP3`/`IMAP`) |
| -------------- | ---- | ------------------------ | ---- | ------------------ | ---- | --------------------------- | ---- | ----------------------- |

But SMTP has two disadvantages inherent to the network protocol:

- The first is that sending an email using SMTP does not return a usable delivery confirmation. Although the specifications of the protocol provide for this type of notification, its formatting is not specified by default, so that usually only an English-language error message, including the header of the undelivered message, is returned.
- Users are not authenticated when a connection is established, and the sender of an email is therefore unreliable. As a result, open SMTP relays are often misused to send spam en masse. The originators use arbitrary fake sender addresses for this purpose to not be traced (mail spoofing). Today, many different security techniques are used to prevent the misuse of SMTP servers. For example, suspicious emails are rejected or moved to quarantine (spam folder). For example, responsible for this are the identification protocol [DomainKeys](http://dkim.org/) (`DKIM`), the [Sender Policy Framework](https://dmarcian.com/what-is-spf/) (`SPF`).

For this purpose, an extension for SMTP has been developed called `Extended SMTP` (`ESMTP`). When people talk about SMTP in general, they usually mean ESMTP. ESMTP uses TLS, which is done after the `EHLO` command by sending `STARTTLS`. This initializes the SSL-protected SMTP connection, and from this moment on, the entire connection is encrypted, and therefore more or less secure. Now [AUTH PLAIN](https://www.samlogic.net/articles/smtp-commands-reference-auth.htm) extension for authentication can also be used safely.

### 2.5.1 Telnet - HELO/EHLO

To interact with the SMTP server, we can use the `telnet` tool to initialize a TCP connection with the SMTP server. The actual initialization of the session is done with the command mentioned above, `HELO` or `EHLO`.

<img src="assets/image-20250714222915812.png" alt="image-20250714222915812" style="width:67%;" />

### 2.5.2 Telnet - VRFY

The command `VRFY` can be used to enumerate existing users on the system. However, this does not always work. Depending on how the SMTP server is configured, the SMTP server may issue `code 252` and confirm the existence of a user that does not exist on the system. A list of all SMTP response codes can be found [here](https://serversmtp.com/smtp-error/).

<img src="assets/image-20250714223154280.png" alt="image-20250714223154280" style="width:50%;" />

### 2.5.3 Send an Email

![image-20250714223706766](assets/image-20250714223706766.png)

### 2.5.4 Footprinting the Service

The default Nmap scripts include `smtp-commands`, which uses the `EHLO` command to list all possible commands that can be executed on the target SMTP server.

<img src="assets/image-20250714225128995.png" alt="image-20250714225128995" style="width:67%;" />

However, we can also use the [smtp-open-relay](https://nmap.org/nsedoc/scripts/smtp-open-relay.html) NSE script to identify the target SMTP server as an open relay using 16 different tests. If we also print out the output of the scan in detail, we will also be able to see which tests the script is running.

<img src="assets/image-20250714225152934.png" alt="image-20250714225152934" style="width:67%;" />

<img src="assets/image-20250714230628545.png" alt="image-20250714230628545" style="width:50%;" />































