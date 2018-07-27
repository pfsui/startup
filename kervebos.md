# 环境

两台装centos7的虚拟机即可。

kerberos服务器端与客户端各一台

（本文档推荐使用Typora软件观看）

# 1.kerberos服务器端配置

## 1.1安装配置Kerberos Server

```bash
[root@localhost ~]# yum install krb5-server krb5-libs krb5-auth-dialog -y
[root@localhost ~]# vi /var/kerberos/krb5kdc/kdc.conf
[kdcdefaults]
kdc_ports = 88
kdc_tcp_ports = 88

[realms]
EXAMPLE.COM = {
#master_key_type = aes256-cts #由于，JAVA使用aes256-cts验证方式需要安装额外的jar包，更多参考2.2.9关于AES-256加密：。推荐不使用。
acl_file = /var/kerberos/krb5kdc/kadm5.acl #标注了admin的用户权限。文件格式是Kerberos_principal permissions [target_principal] [restrictions]支持通配符等。
dict_file = /usr/share/dict/words
admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab #KDC进行校验的keytab（密钥表）。后文会提及如何创建。
max_renewable_life = 7d
supported_enctypes = aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
#supported_enctypes表示支持的校验方式。注意把aes256-cts去掉。
}
```

 

```bash
[root@localhost ~]# vi /etc/krb5.conf
# Configuration snippets may be placed in this directory as well
includedir /etc/krb5.conf.d/

[logging]
default = FILE:/var/log/krb5libs.log #表示server端的日志的打印位置
kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log

[libdefaults]
dns_lookup_realm = false
ticket_lifetime = 24h #表明凭证生效的时限，一般为24小时。
renew_lifetime = 7d #表明凭证最长可以被延期的时限，一般为一个礼拜。当凭证过期之后，对安全认证的服务的后续访问则会失败。
forwardable = true
rdns = false
default_realm = EXAMPLE.COM #默认的realm，必须跟要配置的realm的名称一致。
default_ccache_name = KEYRING:persistent:%{uid}
# udp_preference_limit = 1 禁止使用udp可以防止一个Hadoop中的错误

[realms] #列举使用的realm。
EXAMPLE.COM = {
kdc = kerberos.example.com #代表要kdc的位置。格式是 机器:端口
admin_server = kerberos.example.com #代表admin的位置。格式是机器:端口
}

[domain_realm]
.example.com = EXAMPLE.COM
example.com = EXAMPLE.COM
```

 

## 1.2创建/初始化Kerberos database

```bash
[root@localhost ~]# /usr/sbin/kdb5_util create -s -r EXAMPLE.COM
[root@localhost krb5kdc]# ll /var/kerberos/krb5kdc
总用量 24
-rw-------. 1 root root 22 12月 7 2016 kadm5.acl
-rw-------. 1 root root 459 8月 23 10:45 kdc.conf
-rw-------. 1 root root 8192 8月 23 10:46 principal
-rw-------. 1 root root 8192 8月 23 10:42 principal.kadm5
-rw-------. 1 root root 0 8月 23 10:42 principal.kadm5.lock
-rw-------. 1 root root 0 8月 23 10:46 principal.ok
[-s]表示生成stash file，并在其中存储master server key（krb5kdc）；还可以用[-r]来指定一个realm name —— 当krb5.conf中定义了多个realm时才是必要的。
如果需要重建数据库，将该目录下的principal相关的文件删除即可，其它两个不要删除
在此过程中，我们会输入database的管理密码。这里设置的密码一定要记住，如果忘记了，就无法管理Kerberos server，密码是test
```

## 1.3 添加database administrator数据库管理员

我们需要为Kerberos database添加administrative principals (即能够管理database的principals安全个体) —— 至少要添加1个principal来使得Kerberos的管理进程kadmind能够在网络上与程序kadmin进行通讯。

```bash
[root@localhost ~]# /usr/sbin/kadmin.local -q "addprinc admin/admin"
Authenticating as principal root/admin@EXAMPLE.COM with password.
WARNING: no policy specified for admin/admin@EXAMPLE.COM; defaulting to no policy
Enter password for principal "admin/admin@EXAMPLE.COM": testtest
Re-enter password for principal "admin/admin@EXAMPLE.COM": testtest
Principal "admin/admin@EXAMPLE.COM" created.
[root@localhost ~]#

添加佣有管理员权限的管理员ryan密码为123456
[root@localhost ~]# kadmin.local
Authenticating as principal root/admin@EXAMPLE.COM with password.
kadmin.local: addprinc ryan/admin #添加ryan/admin
WARNING: no policy specified for ryan/admin@EXAMPLE.COM; defaulting to no policy
Enter password for principal "ryan/admin@EXAMPLE.COM": 123456
Re-enter password for principal "ryan/admin@EXAMPLE.COM": 123456
Principal "ryan/admin@EXAMPLE.COM" created.
kadmin.local: listprincs #查看有多少
K/M@EXAMPLE.COM
admin/admin@EXAMPLE.COM
kadmin/admin@EXAMPLE.COM
kadmin/changepw@EXAMPLE.COM
kadmin/localhost@EXAMPLE.COM
kiprop/localhost@EXAMPLE.COM
krbtgt/EXAMPLE.COM@EXAMPLE.COM
ryan/admin@EXAMPLE.COM
kadmin.local: delprinc ryan/admin #删除ryan/admin 不能只是ryan
Are you sure you want to delete the principal "ryan/admin@EXAMPLE.COM"? (yes/no): yes
Principal "ryan/admin@EXAMPLE.COM" deleted.
Make sure that you have removed this principal from all ACLs before reusing.
kadmin.local: modprinc -maxrenewlife 1week ryan/admin@EXAMPLE.COM #修改renewlife为7天
Principal "ryan/admin@EXAMPLE.COM" modified.
kadmin.local: exit

以下命令还不能使用：
[root@localhost ~]# kadmin
-bash: kadmin: 未找到命令
```

## 1.4 为database administrator设置ACL权限

```bash
[root@localhost krb5kdc]# cat /var/kerberos/krb5kdc/kadm5.acl
*/admin@EXAMPLE.COM *
# 代表名称匹配*/admin@EXAMPLE.COM的，都认为是admin，权限是 *。代表全部权限。
在KDC上我们需要编辑acl文件来设置权限，该acl文件的默认路径是 /var/kerberos/krb5kdc/kadm5.acl（也可以在文件kdc.conf中修改）。Kerberos的kadmind daemon会使用该文件来管理对Kerberos database的访问权限。对于那些可能会对pincipal产生影响的操作，acl文件也能控制哪些principal能操作哪些其他pricipals。


```

# 1.5 在master KDC启动Kerberos daemons

```bash
[root@localhost krb5kdc]# systemctl status krb5kdc
● krb5kdc.service - Kerberos 5 KDC
Loaded: loaded (/usr/lib/systemd/system/krb5kdc.service; disabled; vendor preset: disabled)
Active: inactive (dead)
[root@localhost krb5kdc]# systemctl start krb5kdc
[root@localhost krb5kdc]# systemctl enable krb5kdc

[root@localhost krb5kdc]# systemctl status kadmin
● kadmin.service - Kerberos 5 Password-changing and Administration
Loaded: loaded (/usr/lib/systemd/system/kadmin.service; disabled; vendor preset: disabled)
Active: inactive (dead)
[root@localhost krb5kdc]# systemctl start kadmin
[root@localhost krb5kdc]# systemctl enable kadmin

[root@localhost krb5kdc]# tail -f /var/log/krb5kdc.log
otp: Loaded
8月 23 11:17:20 localhost.localdomain krb5kdc[4914](info): setting up network...
8月 23 11:17:20 localhost.localdomain krb5kdc[4914](info): listening on fd 9: udp 0.0.0.0.88 (pktinfo)
krb5kdc: setsockopt(10,IPV6_V6ONLY,1) worked
8月 23 11:17:20 localhost.localdomain krb5kdc[4914](info): listening on fd 10: udp ::.88 (pktinfo)
krb5kdc: setsockopt(11,IPV6_V6ONLY,1) worked
8月 23 11:17:20 localhost.localdomain krb5kdc[4914](info): listening on fd 12: tcp 0.0.0.0.88
8月 23 11:17:20 localhost.localdomain krb5kdc[4914](info): listening on fd 11: tcp ::.88
8月 23 11:17:20 localhost.localdomain krb5kdc[4914](info): set up 4 sockets
8月 23 11:17:20 localhost.localdomain krb5kdc[4915](info): commencing operation
[root@localhost krb5kdc]# tail -f /var/log/kadmind.log
8月 23 11:17:32 localhost.localdomain kadmind[4940](info): listening on fd 10: udp ::.464 (pktinfo)
kadmind: setsockopt(11,IPV6_V6ONLY,1) worked
8月 23 11:17:32 localhost.localdomain kadmind[4940](info): listening on fd 12: tcp 0.0.0.0.464
8月 23 11:17:32 localhost.localdomain kadmind[4940](info): listening on fd 11: tcp ::.464
8月 23 11:17:32 localhost.localdomain kadmind[4940](info): listening on fd 13: rpc 0.0.0.0.749
kadmind: setsockopt(14,IPV6_V6ONLY,1) worked
8月 23 11:17:32 localhost.localdomain kadmind[4940](info): listening on fd 14: rpc ::.749
8月 23 11:17:32 localhost.localdomain kadmind[4940](info): set up 6 sockets
8月 23 11:17:32 localhost.localdomain kadmind[4941](info): Seeding random number generator
8月 23 11:17:32 localhost.localdomain kadmind[4941](info): starting
```

现在KDC已经在工作了。这两个daemons将会在后台运行，可以查看它们的日志文件（/var/log/krb5kdc.log 和 /var/log/kadmind.log）。
可以通过客户端命令kinit来检查这两个daemons是否正常工作。

 

# 2.客户端操作

## 2.1安装客户端

```bash
[root@localhost ~]# yum install -y krb5-workstation krb5-libs krb5-auth-dialog
```

## 2.2 配置krb5.conf

配置这些主机上的/etc/krb5.conf，这个文件的内容与KDC服务器中的文件保持一致即可。

## 2.3验证后登录

登录到管理员账户: 如果在本机上，可以通过kadmin.local直接登录。其它机器的，先使用kinit进行验证。

```bash
验证：
[root@localhost ~]# kinit ryan/admin@EXAMPLE.COM
Password for ryan/admin@EXAMPLE.COM: 123456
登录：
[root@localhost ~]# kadmin
Authenticating as principal ryan/admin@EXAMPLE.COM with password.
Password for ryan/admin@EXAMPLE.COM: 
kadmin: list_principals #列出所有帐户
K/M@EXAMPLE.COM
admin/admin@EXAMPLE.COM
kadmin/admin@EXAMPLE.COM
kadmin/changepw@EXAMPLE.COM
kadmin/localhost@EXAMPLE.COM
kiprop/localhost@EXAMPLE.COM
krbtgt/EXAMPLE.COM@EXAMPLE.COM
ryan/admin@EXAMPLE.COM
kadmin: 
查看当前的认证用户：
[root@localhost ~]# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: ryan/admin@EXAMPLE.COM

Valid starting Expires Service principal
2017-08-23T14:01:32 2017-08-24T14:01:32 krbtgt/EXAMPLE.COM@EXAMPLE.COM
renew until 2017-08-30T14:01:32
[root@localhost ~]# 
```

## 2.4 创建keytab

```bash
[root@localhost tmp]# mkdir -p /var/kerberos/krb5kdc/
[root@localhost tmp]# kinit ryan/admin@EXAMPLE.COM
[root@localhost tmp]# kadmin
#创建key table(密钥表)命令(第一种)
kadmin: xst -k /var/kerberos/krb5kdc/kadm5.keytab ryan/admin@EXAMPLE.COM 
Entry for principal ryan/admin@EXAMPLE.COM with kvno 25, encryption type aes128-cts-hmac-sha1-96 added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 25, encryption type des3-cbc-sha1 added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 25, encryption type arcfour-hmac added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 25, encryption type camellia256-cts-cmac added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 25, encryption type camellia128-cts-cmac added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 25, encryption type des-hmac-sha1 added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 25, encryption type des-cbc-md5 added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
#创建key table(密钥表)命令(第二种)
kadmin: ktadd -k /var/kerberos/krb5kdc/kadm5.keytab ryan/admin@EXAMPLE.COM
Entry for principal ryan/admin@EXAMPLE.COM with kvno 24, encryption type aes128-cts-hmac-sha1-96 added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 24, encryption type des3-cbc-sha1 added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 24, encryption type arcfour-hmac added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 24, encryption type camellia256-cts-cmac added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 24, encryption type camellia128-cts-cmac added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 24, encryption type des-hmac-sha1 added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
Entry for principal ryan/admin@EXAMPLE.COM with kvno 24, encryption type des-cbc-md5 added to keytab WRFILE:/var/kerberos/krb5kdc/kadm5.keytab.
kadmin: 
查看keytab里添加了哪些密钥（每个帐户有不同类型的）
[root@localhost ~]# klist -e -k -t /var/kerberos/krb5kdc/kadm5.keytab
```

## 2.5使用密钥表的方式登录（无需输入密码）

```bash
使用之前的密码方式登录，会报密码错误，因为生成密钥表的时候，会重新生成一个随机密钥，然后再写入keytab密钥表文件中。以下kinit验证步骤可省略：
[root@localhost ~]# kinit -kt /var/kerberos/krb5kdc/kadm5.keytab ryan/admin@EXAMPLE.COM
[root@localhost ~]# kadmin -kt /var/kerberos/krb5kdc/kadm5.keytab -p ryan/admin@EXAMPLE.COM
Authenticating as principal ryan/admin@EXAMPLE.COM with keytab /var/kerberos/krb5kdc/kadm5.keytab.
kadmin: ?
```

## 2.6删除当前认证缓存

```bash
[root@localhost ~]# kdestroy
```

## 2.7延长凭证过期时间

```bash
[root@localhost ~]# kinit -kt /var/kerberos/krb5kdc/kadm5.keytab ryan/admin@EXAMPLE.COM
[root@localhost ~]# kinit -R #延长凭证过期时间
kinit: KDC can‘t fulfill requested option while renewing credentials
注：提示无法延长凭证过期时间，可能是因为renewlife参数设置为0day了。

[root@localhost ~]# kadmin.local
kadmin.local: modprinc -maxrenewlife 1week ryan/admin@EXAMPLE.COM
Principal "ryan/admin@EXAMPLE.COM" modified.

[root@localhost ~]# kinit -kt /var/kerberos/krb5kdc/kadm5.keytab ryan/admin@EXAMPLE.COM
[root@localhost ~]# kinit -R 
[root@localhost ~]# #没有返回信息，说明已经成功延长凭证过期时间了
```

 

 

# 3.常见错误

```bash
[root@localhost ~]# /usr/sbin/kdb5_util create -s -r EXAMPLE.COM
kdb5_util: Required parameters in kdc.conf missing while initializing the Kerberos admin interface
配置文件中的supported_enctypes的某个加密类型不可用。


```

 

```bash
[root@localhost ~]# /usr/sbin/kadmin.local -q "addprinc admin/admin"
Authenticating as principal root/admin@EXAMPLE.COM with password.
kadmin.local: Cannot find master key record in database while initializing kadmin.local interface
需要重新运行创建kerberos数据库的命令，即/usr/sbin/kdb5_util create -s -r EXAMPLE.COM
```

 

```bash
[root@localhost ~]# kinit ryan/admin@EXAMPLE.COM
kinit: Cannot contact any KDC for realm ‘EXAMPLE.COM‘ while getting initial credentials
KDC服务器的防火墙没关，或者KDC服务器服务没启动
```

 

```bash
[root@localhost ~]# kinit ryan/admin@EXAMPLE.COM
Password for ryan/admin@EXAMPLE.COM: 
kinit: Password incorrect while getting initial credentials
提示密码错误，或者是因为执行创建密钥表的操作，重新生成随机密钥，写入密钥表文件中。
```

 

 

4.参考文章

```bash
http://dongxicheng.org/mapreduce/hadoop-kerberos-introduction/
http://blog.csdn.net/wulantian/article/details/42418231
http://idior.cnblogs.com/archive/2006/03/20/354027.html
```

 

 

```bash
http://docs.oracle.com/cd/E24847_01/html/819-7061/setup-9.html
http://www.cnblogs.com/xiaodf/p/5968178.html
```
