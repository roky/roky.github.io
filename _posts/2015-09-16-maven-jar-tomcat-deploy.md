---
layout: post
title:  "maven管理项目构件"
date:   2015-09-16 30:06:06
categories: test
excerpt: 公司只有内网
---

* content
{:toc}

最近公司对老框架技术改造，基于之前项目在开发、发布过程中出现的问题，决定采用maven来管理项目

## 搭建本地maven仓库（Center Repository）

![](/images/maven/11.png)

---

## 管理未代理第三方库(3rd Repository）)

![](/images/maven/21.png)
![](/images/maven/22.png)
![](/images/maven/23.png)

---

## 公司类库上传到服务器仓库（Local Repository））

### 1、建立maveng工程  artifactid选择maven-archeype-webapp

###  2、在Maven目录中设置settings.xml文件中设置上传的权限

![](/images/maven/31.png)
![](/images/maven/32.png)

### 3、在工程项目的pom.xml文件中设置

![](/images/maven/33.png)

### 4、发布到服务器仓库中,运行

     mvn deploy

### 5、查看是否成功

![](/images/maven/34.png)

---

## 通过maven实现远程每日测试构件

### 1、在settings.xml设置权限

![](/images/maven/41.png)

### 2、在tomcat的tomcat-users.xml配置用户权限

![](/images/maven/42.png)

### 3、发布文件部署到tomcat中，在pom.xml在增加部署信息

![](/images/maven/43.png)

### 4、运行 

     mvn clean tomcat7:redeploy