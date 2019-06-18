---
layout:     post
title:      gitlab+jenkins+docker+kubernetes一键发布
subtitle:   gitlab+jenkins+docker+kubernetes一键发布
date:       2019-06-18
author:     tianchi yuan
header-img: img/in-post/post-bg-unix-linux.jpg
catalog: true
tags:
    - kubernetes
---

# 整体逻辑梳理：
使用上了kubernetes后每次发布项目的新版本都会涉及到项目镜像的打包、上传、更新pod的镜像等步骤，感觉有些繁琐，也不符合我们"AUTOMATE EVERYTHING"的理念。
然后我进行了如下几个步骤后，实现了项目到kubernetes的一键发布
1. 创建Dockerfile,添加到gitlab的项目的根目录下
2. 在jenkins所在服务器安装docker，kubectl
3. 编写自动打包、上传、发布镜像的脚本

# 详细步骤
## 1.编写Dockerfile,添加到gitlab项目分支下
![avatar](https://yuantianchi.github.io/posts_image/kubernetes/Dockerfile.png)  

这是根据我项目的实际情况编写的Dockerfile
```
FROM tomcat:7.0.94-jre8

ENV CATALINA_HOME /usr/local/tomcat
ENV JAVA_OPTS="-Dglobal.config.path=/data/config"

RUN mkdir -p /data/config \  
    && mkdir -p /data/logs \
    && rm -rf $CATALINA_HOME/webapps/ROOT
COPY ./ofc-web/target/ofc-web*.war $CATALINA_HOME/webapps/ofc.war

EXPOSE 8080
CMD ["catalina.sh", "run"]
```
## 2.在jenkins所在服务器安装docker，kubectl
1. 在jenkins的服务器上安装docker并配置其有权限能上传镜像到你的镜像仓库（我这里使用的是ucloud的docker镜像仓库）

2. 安装kuberctl添加config使此台机器能通过kubectl命令操作kubernetes集群

## 3.编写自动打包、上传、发布镜像的脚本
首先配置jenkins拉取gitlab项目代码功能，然后编写和添加如下自动发布脚本

### 脚本说明：  
      1. 项目每个版本都会有Git为其分配的唯一hash值，我是用这个hash值来作为项目镜像的tag
      2. 我这里为了避免jenkins机器的docker镜像过多，在脚本中添加了删掉老版本的镜像的功能
      3. 脚本是根据我的环境实际情况所编写的，您需要根据实际情况修改
      
```
project_name="ofc"
git_id=`git rev-parse --short HEAD`
image_id=`sudo docker images | grep ${git_id} | awk '{ if($1=="$project_name"){ print $3}}' |head -n 1`
image_id_list=`sudo docker images | awk '{if ($1=="'$project_name'") print $3}' | sort -u`

if [ -n "$image_id" ];then
   echo "id为${image_id}的镜像已存在" 
else
   echo "开始打包镜像"
   sudo docker build -t ${project_name}:${git_id} . 
   sudo docker tag -f ${project_name}:${git_id} uhub.service.ucloud.cn/topideal_hub/${project_name}:${git_id}
fi
echo "push镜像到ucloud仓库"
sudo docker push uhub.service.ucloud.cn/topideal_hub/${project_name}:${git_id}
echo "打包和push镜像完成"

if [ -n "${image_id_list}" ];then
   for id in ${image_id_list} ;do
     if [ "$id" != "$image_id" ];then
       echo "删除id为${id}的老版本的镜像"
       sudo docker rmi -f ${id}
     fi
   done
fi
echo "构建完成,当前tag为${git_id}"
echo "开始发布到kubernets中"
/usr/local/bin/kubectl set image deployment/${project_name} ${project_name}=uhub.service.ucloud.cn/topideal_hub/${project_name}:${git_id}
echo "已成功发布，使用kubectl 查看更新状态"

```

然后点击jenkins构建后,就可以自动的完成项目的镜像打包、上传、发布了
