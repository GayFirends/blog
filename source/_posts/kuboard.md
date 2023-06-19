---
title: 在K8S上搭建Kuboard面板
date: 2023-06-19 21:45:00
tags: [k8s,面板,kuboard,linux,docker]
---

上次我们搭建了K8S的dashboard，然后近期我在上课的时候发现了一个叫做kuboard的项目，是一个K8S的管理面板，比官方的dashboard要好用（主要是有中文！！！），废话不多说我们直接搞起！

# 开始搭建

## 1）开启已经搭建好的K8S虚拟机

  目前我这边使用的是主节点+一台单节点集群。

## 2）使用hostPath持久化方案安装kuboard

​	运行如下命令进行安装

```bash
sudo docker run -d \
  --restart=unless-stopped \
  --name=kuboard \
  -p 80:80/tcp \
  -p 10081:10081/tcp \
  -e KUBOARD_ENDPOINT="http://内网IP:80" \
  -e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
  -v /root/kuboard-data:/data \
  swr.cn-east-2.myhuaweicloud.com/kuboard/kuboard:v3
```

![image-20230619212726679](https://s2.loli.net/2023/06/19/BgRHI1na7yYrX2A.png)

## 3）访问Kuboard

- 在浏览器中打开链接 `http://your-node-ip-address:80`
- 输入初始用户名和密码，并登录
  - 用户名： `admin`
  - 密码： `Kuboard123`![image-20230619213027603](https://s2.loli.net/2023/06/19/pYa4wt3Ub7zm6Gq.png)

## 4）添加集群

单击添加集群-->选择agent然后按照如下设置：

![image-20230619213309803](https://s2.loli.net/2023/06/19/y8mSQ1NbWCBit69.png)

注意：务必选择![image-20230619213327815](https://s2.loli.net/2023/06/19/ojZQD7cUIG9JNkl.png)此镜像，因为Dokcer官方镜像库目前在中国大陆无法正常使用，所以选择此选项。

然后根据提示在终端输入如下命令：![image-20230619214129605](https://s2.loli.net/2023/06/19/qmyakEuDNSov4wT.png)

![image-20230619214157364](https://s2.loli.net/2023/06/19/4FWVtfSOZpEmJkG.png)

# 5）结束

![image-20230619214246582](https://s2.loli.net/2023/06/19/ZKzYaCtiBLV42n7.png)

这样就搭建完成了！接下来我会慢慢更新关于k8s的内容，如有纰漏，麻烦在评论区指出，谢谢！
