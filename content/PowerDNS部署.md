---
title: "PowerDNS部署"
date: 2018-12-26T14:00:56+08:00
draft: false
---

# PowerDNS
PowerDNS提供两种服务：
- 权威域名服务
- 转发域名服务

下面进行两种服务的安装部署过程
# PowerDNS Authoritative Server

## 安装
```
yum install -y pdns
yum install -y pdns-backend-mysql
```
[更多后端支持类型](https://doc.powerdns.com/authoritative/backends/index.html)


## 修改配置
vim /etc/pdns/pdns.conf
```
launch=gmysql
gmysql-host=10.2.4.118
gmysql-user=pdns
gmysql-dbname=pdns
gmysql-password=mysecretpassword
```

## 初始化数据库
```
MariaDB [(none)]> create database pdns;
MariaDB [(none)]> use pdns
MariaDB [pdns]> CREATE USER 'pdns'@'localhost' IDENTIFIED BY 'mysecretpassword';
MariaDB [pdns]> grant all on pdns.* to pdns@'%' identified by 'mysecretpassword';
#创建表
CREATE TABLE domains (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  VARCHAR(6) NOT NULL,
  notified_serial       INT UNSIGNED DEFAULT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE UNIQUE INDEX name_index ON domains(name);


CREATE TABLE records (
  id                    BIGINT AUTO_INCREMENT,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(64000) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  disabled              TINYINT(1) DEFAULT 0,
  ordername             VARCHAR(255) BINARY DEFAULT NULL,
  auth                  TINYINT(1) DEFAULT 1,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX nametype_index ON records(name,type);
CREATE INDEX domain_id ON records(domain_id);
CREATE INDEX ordername ON records (ordername);


CREATE TABLE supermasters (
  ip                    VARCHAR(64) NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (ip, nameserver)
) Engine=InnoDB CHARACTER SET 'latin1';


CREATE TABLE comments (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  comment               TEXT CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX comments_name_type_idx ON comments (name, type);
CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);


CREATE TABLE domainmetadata (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  kind                  VARCHAR(32),
  content               TEXT,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX domainmetadata_idx ON domainmetadata (domain_id, kind);


CREATE TABLE cryptokeys (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  flags                 INT NOT NULL,
  active                BOOL,
  content               TEXT,
  PRIMARY KEY(id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE INDEX domainidindex ON cryptokeys(domain_id);


CREATE TABLE tsigkeys (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);

#插入一些测试数据
INSERT INTO domains (name, type) values ('example.com', 'NATIVE');
INSERT INTO records (domain_id, name, content, type,ttl,prio)
VALUES (1,'example.com','localhost admin.example.com 1 10380 3600 604800 3600','SOA',86400,NULL);
INSERT INTO records (domain_id, name, content, type,ttl,prio)
VALUES (1,'example.com','dns-us1.powerdns.net','NS',86400,NULL);
INSERT INTO records (domain_id, name, content, type,ttl,prio)
VALUES (1,'example.com','dns-eu1.powerdns.net','NS',86400,NULL);
INSERT INTO records (domain_id, name, content, type,ttl,prio)
VALUES (1,'www.example.com','192.0.2.10','A',120,NULL);
INSERT INTO records (domain_id, name, content, type,ttl,prio)
VALUES (1,'mail.example.com','192.0.2.12','A',120,NULL);
INSERT INTO records (domain_id, name, content, type,ttl,prio)
VALUES (1,'localhost.example.com','127.0.0.1','A',120,NULL);
INSERT INTO records (domain_id, name, content, type,ttl,prio)
VALUES (1,'example.com','mail.example.com','MX',120,25);
```
## 启动服务
```
systemctl start pdns.service
```
## 验证服务
```
[root@pdns ~]# dig +short www.example.com @127.0.0.1
192.0.2.10
```
至此：PowerDNS权威域名服务部署完成

# PowerDNS Recursor
## 安装
```
yum install -y pdns-recursor
```
## 配置
```
cat /etc/pdns-recursor/recursor.conf
setuid=pdns-recursor
setgid=pdns-recursor
local-address=0.0.0.0
local-port=53
forward-zones-file=/etc/pdns-recursor/forward-zones

[root@powerdns-master ~]# cat /etc/pdns-recursor/forward-zones
bldz.local=10.2.3.123:1053,10.2.3.124:1053
pro.testin.cn=10.2.3.123:1053,10.2.3.124:1053
```
## 启动
```
systemctl start pdns-recursor.service
```


## PowerDNS扩展

### 权威域名服务和转发域名服务适用场景:

- 权威域名服务：公司维护自己一套域名解析，例如：bldz.com，自己购买两台DNS服务器，然后在域名的基本信息中进行修改地址，然后阿里云就会向.com域名解析更改为自己DNS地址。


- 转发域名服务：负责发出其他请求以满足客户端的DNS查询。
- 当公司内部有一套DNS，也需要使用外部DNS的时候，两种服务都需要安装。

## PowerDNS的web服务
修改配置：
vim /etc/pdns/pdns.conf
```
daemon=no
guardian=no
launch=gmysql
gmysql-host=10.2.4.118
gmysql-user=pdns
gmysql-dbname=pdns
gmysql-password=mysecretpassword
setgid=pdns
setuid=pdns
local-address=0.0.0.0
local-port=1053
webserver=yes
webserver-address=0.0.0.0
webserver-allow-from=0.0.0.0/0,::/0
webserver-password=123456
webserver-port=8081
webserver-print-arguments=no

#浏览器访问：
http://dnsserver:8081
user:admin
passwd:123456
```
## PowerDNS的UI界面
[所有UI地址](https://github.com/PowerDNS/pdns/wiki/WebFrontends)

[推荐UI界面](https://github.com/ngoduykhanh/PowerDNS-Admin)