---
layout: post
title: Centos7 搭建 OpenLDAP
date: 2022-03-24 11:28:00
permalink: /tool/openldap/guide.html
categories:
- 工具
tags:
- OpenLDAP
- 部署记录
---

## Centos7 搭建 OpenLDAP

基本转载自：[Centos7 搭建openldap完整详细教程(真实可用)](https://blog.csdn.net/weixin_41004350/article/details/89521170) 

版本说明：

CentOS 7.9

[OpenLDAP](https://www.openldap.org/software/) 2.4.44

<!--more-->

### 安装 OpenLDAP

```bash
# yum 安装相关包
yum install -y openldap openldap-clients openldap-servers

# 复制一个默认配置到指定目录下,并授权，这一步一定要做，然后再启动服务，不然生产密码时会报错
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown -R ldap. /var/lib/ldap/DB_CONFIG

# 启动服务
systemctl start slapd
systemctl enable slapd

# 查看状态
systemctl status slapd
```

### 修改 OpenLDAP 配置

> 从 OpenLDAP 2.4.23 版本开始，所有配置都保存在 `/etc/openldap/slapd.d` 目录下的 `cn=config` 目录内，不再使用 `slapd.conf` 作为配置文件。配置文件的后缀为 `ldif`，且每个配置文件都是通过命令自动生成的，任意打开一个配置文件，在开头都会有一行注释，说明此为自动生成的文件，请勿编辑，使用 `ldapmodify` 命令进行修改。
>
> ```yaml
> # AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
> ```



OpenLDAP 有三个命令用于修改配置文件，分别为：

- `ldapadd` ：添加
- `ldapmodify`：修改
- `ldapdelete`：删除

需要修改或增加配置时，需要先写一个 `ldif` 后缀的配置文件，然后通过命令将配置更新到 `slapd.d` 目录下的配置文件中去，完整的配置过程如下：

```bash
# 生成管理员密码,记录下这个密码，后面需要用到
slappasswd -s nodetower
{SSHA}pAX58EOnqE1NzrZQKVt9ceBcHly9Bf8m
 
# 新增修改密码文件,ldif为后缀，文件名随意，不要在/etc/openldap/slapd.d/目录下创建类似文件
# 生成的文件为需要通过命令去动态修改ldap现有配置，如下，我在家目录下，创建文件
cd ~
vim changepwd.ldif
----------------------------------------------------------------------
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}pAX58EOnqE1NzrZQKVt9ceBcHly9Bf8m
----------------------------------------------------------------------
# 这里解释一下这个文件的内容：
# 第一行执行配置文件，这里就表示指定为 cn=config/olcDatabase={0}config 文件。你到/etc/openldap/slapd.d/目录下就能找到此文件
# 第二行 changetype 指定类型为修改
# 第三行 add 表示添加 olcRootPW 配置项
# 第四行指定 olcRootPW 配置项的值
# 在执行下面的命令前，你可以先查看原本的olcDatabase={0}config文件，里面是没有olcRootPW这个项的，执行命令后，你再看就会新增了olcRootPW项，而且内容是我们文件中指定的值加密后的字符串
 
# 执行命令，修改ldap配置，通过-f执行文件
ldapadd -Y EXTERNAL -H ldapi:/// -f changepwd.ldif
```

执行修改命令后，有如下输出则为正常：

<img src="https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220324115530453.png" alt="image-20220324115530453" style="zoom: 50%;" />

切记不能直接修改/etc/openldap/slapd.d/目录下的配置。

继续进行配置

```bash
# 我们需要向 LDAP 中导入一些基本的 Schema。这些 Schema 文件位于 /etc/openldap/schema/ 目录中，schema控制着条目拥有哪些对象类和属性，可以自行选择需要的进行导入，
# 依次执行下面的命令，导入基础的一些配置,我这里将所有的都导入一下，其中core.ldif是默认已经加载了的，不用导入
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/collective.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/corba.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/duaconf.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/dyngroup.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/java.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/misc.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/pmi.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/ppolicy.ldif
 
 
 
# 修改域名，新增changedomain.ldif, 这里我自定义的域名为 roiquery.com，管理员用户账号为admin。
# 如果要修改，则修改文件中相应的dc=roiquery,dc=com为自己的域名
vim changedomain.ldif
-------------------------------------------------------------------------
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=admin,dc=roiquery,dc=com" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=roiquery,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=roiquery,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}pAX58EOnqE1NzrZQKVt9ceBcHly9Bf8m

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by dn="cn=admin,dc=roiquery,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=admin,dc=roiquery,dc=com" write by * read
-------------------------------------------------------------------------
 
# 执行命令，修改配置
ldapmodify -Y EXTERNAL -H ldapi:/// -f changedomain.ldif
```

这里有 5 个修改，所以执行会输出 5 行表示成功。

<img src="https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220324141915660.png" alt="image-20220324141915660" style="zoom: 50%;" />

启用 memberof 功能

```bash
# 新增add-memberof.ldif, #开启memberof支持并新增用户支持memberof配置
vim add-memberof.ldif
-------------------------------------------------------------
dn: cn=module{0},cn=config
cn: modulle{0}
objectClass: olcModuleList
objectclass: top
olcModuleload: memberof.la
olcModulePath: /usr/lib64/openldap

dn: olcOverlay={0}memberof,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfUniqueNames
olcMemberOfMemberAD: uniqueMember
olcMemberOfMemberOfAD: memberOf
-------------------------------------------------------------
 
 
# 新增refint1.ldif文件
vim refint1.ldif
-------------------------------------------------------------
dn: cn=module{0},cn=config
add: olcmoduleload
olcmoduleload: refint
-------------------------------------------------------------
 
 
# 新增refint2.ldif文件
vim refint2.ldif
-------------------------------------------------------------
dn: olcOverlay=refint,olcDatabase={2}hdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: refint
olcRefintAttribute: memberof uniqueMember  manager owner
-------------------------------------------------------------
 
 
# 依次执行下面命令，加载配置，顺序不能错
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f add-memberof.ldif
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f refint1.ldif
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f refint2.ldif
```

结果：

<img src="https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220324142144302.png" alt="image-20220324142144302" style="zoom:50%;" />

到此，配置修改完了，在上述基础上，我们来创建一个叫做 roiquery company 的组织，并在其下创建一个 admin 的组织角色（该组织角色内的用户具有管理整个 LDAP 的权限）和 People 和 Group 两个组织单元：

```bash
# 新增配置文件
vim base.ldif
----------------------------------------------------------
dn: dc=roiquery,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: ROIQuery Team
dc: roiquery

dn: cn=admin,dc=roiquery,dc=com
objectClass: organizationalRole
cn: admin

dn: ou=People,dc=roiquery,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=roiquery,dc=com
objectClass: organizationalRole
cn: Group
----------------------------------------------------------
 
# 执行命令，添加配置, 这里要注意修改域名为自己配置的域名，然后需要输入上面我们生成的密码
ldapadd -x -D cn=admin,dc=roiquery,dc=com -W -f base.ldif
```

结果：

<img src="https://image-1257603108.cos.ap-guangzhou.myqcloud.com/image-20220324142516132.png" alt="image-20220324142516132" style="zoom:50%;" />

通过以上的所有步骤，我们就设置好了一个 LDAP 目录树：其中基准 dc=roiquery,dc=com 是该树的根节点，其下有一个管理域 cn=admin,dc=roiquery,dc=com 和两个组织单元 ou=People,dc=roiquery,dc=com 及 ou=Group,dc=roiquery,dc=com。
