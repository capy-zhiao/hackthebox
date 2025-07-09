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

## 2 Host Based Enumeration

### 2.1 FTP

21: control, 20: data trans

https://en.wikipedia.org/wiki/List\_of\_FTP\_server\_return\_codes

FTP is a `clear-text` protocol that can sometimes be sniffed if conditions on the network are right.

#### 2.1.1 TFTP

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
* test
