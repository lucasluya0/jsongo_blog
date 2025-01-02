个人开发者从购入一台云主机开始

![img_3.png](img_3.png)
`推荐原因：` 能快速搭建学习编程的基础环境，像mysql，redis等基础服务


## 1. 安装docker

#### 第一步：直接加入镜像源，然后在线安装
```shell
sudo dnf config-manager --add-repo=https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
或者
```shell
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```
如果报错
```shell
Error: Failed to download metadata for repo 'docker-ce-stable': Cannot download repomd.xml: Cannot download repodata/repomd.xml: All mirrors were tried
```
解决方案：进入/etc/yum.repos.d路径下，找到docker-ce.repo文件，把对应 $releasever 修改成自己的Centos版本
![img.png](img.png)

![img_1.png](img_1.png)

#### 第二步： 启动Docker并设置Docker守护进程在系统启动时自动启动，这样可以确保每次系统启动时，Docker服务也会自动启动。
```shell
sudo systemctl start docker
sudo systemctl enable docker
```

##  2. 安装`1panel`
```shell
curl -sSL https://resource.fit2cloud.com/1panel/package/quick_start.sh -o quick_start.sh && sh quick_start.sh
```
![img_2.png](img_2.png)

具体使用可以参考 `https://mp.weixin.qq.com/s/8_81JG1cE1zEmefK870pNw`

参考文档：https://cloud.tencent.com/developer/article/2302184，如有侵权，请联系我删除。

### 如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！