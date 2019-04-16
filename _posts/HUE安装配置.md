# HUE
## 什么是HUE

####

#### 待填坑

####

## 编译HUE
本篇文章选用hue-3.9.0-cdh5.7.0
**安装依赖软件包**

```shell
shell> yum -y install ant asciidoc cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-plain gcc gcc-c++ krb5-devel libffi-devel libxml2-devel libxslt-devel make  mysql mysql-devel openldap-devel python-devel sqlite-devel gmp-devel
```
**编译HUE**
```shell
## 在这之前需要先安装好jdk、maven等环境
## 配置Maven仓库，指向aliyun 
##    <mirror>
##    <id>nexus-aliyun</id>
##    <mirrorOf>central</mirrorOf>
##    <name>Nexus aliyun</name>
##    <url>http://maven.aliyun.com/nexus/content/groups/public</url>
##    </mirror>
##
##
##
shell> wget http://archive.cloudera.com/cdh5/cdh/5/hue-3.9.0-cdh5.7.0.tar.gz
shell> tar zxfv hue-3.9.0-cdh5.7.0.tar.gz
shell> cd hue-3.9.0-cdh5.7.0
shell> make apps

```
## 编辑配置文件

### 编辑Hadoop配置文件

**hdfs-site.xml**

