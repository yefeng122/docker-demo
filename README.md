# Docker安装使用
## 安装docker（或者安装docker-ce）
```sh
yum install -y docker
```
## 安装docker-ce（上面的安装方式有可能版本不是最新的，导致无法使用docker-compose）
```sh
yum install -y yum-utils \
    device-mapper-persistent-data \
    lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce
```
## 启动
```sh
systemctl start docker
```
## 编写Dockerfile
```docker
# 基于哪个镜像
FROM java:8 
# 将本地文件夹挂载到当前容器
VOLUME /tmp
# 复制文件到容器
ADD docker-demo-0.0.1-SNAPSHOT.jar app.jar
RUN bash -c 'touch /app.jar'
# 声明需要暴露的端口
EXPOSE 8888
# 配置容器启动后的命令
ENTRYPOINT ["java","-jar","/app.jar"]
```
## 使用docker build命令构建镜像，注意后面的"."别丢了
```sh
docker build -t bigbaldy/docker-demo .
```
## 启动镜像
```sh
docker run -d -p 8888:8888 bigbaldy/docker-demo
```
## 访问http://127.0.0.1:8888/hello 返回hello kubernetes
## 推送镜像
```sh
docker push bigbaldy/docker-demo //需要输入你在DockerHub上注册的用户密码
```
## 推送镜像到私有库(127.0.0.1)
    - docker run -d -p 5000:5000 registry:2 //使用docker registry2.0搭建私有仓库
    - docker tag bigbaldy/docker-demo 127.0.0.1:5000/docker-demo //修改镜像标签，否则会提示找不到镜像的
    - docker push 127.0.0.1:5000/docker-demo
## 创建私有库(真实IP)
* 生成证书
```sh
mkdir -p ~/certs
cd ~/certs
openssl genrsa -out reg.codesafe.com.key 2048
```
* ⽣成密钥⽂件，会有⼀些信息需要填写，注意“Common Name”要填写“reg.codesafe.com”
```sh
openssl req -newkey rsa:4096 -nodes -sha256 -keyout reg.codesafe.com.key -x509 -days 365 -out reg.codesafe.com.crt
```
* 将生成的证书拷贝到根证书路径
```sh
mkdir -p /etc/docker/certs.d/reg.codesafe.com
cp ~/certs/reg.itmuch.com.crt /etc/docker/certs.d/reg.codesafe.com
```
* 重启docker
```sh
systemctl restart docker
```
* 启动私有库，注意要在~目录下执行
```sh
docker run -d -p 443:5000 \
    --restart=always \
    --name registry \
    -v `pwd`/certs:/certs \
    -v /opt/docker-image:/opt/docker-image \
    -e STORAGE_PATH=/opt/docker-image \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/reg.codesafe.com.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/reg.codesafe.com.key \
    registry:2
```
* 修改镜像标签，否则会提示找不到镜像的
```sh
docker tag bigbaldy/docker-demo reg.codesafe.com/bigbaldy/docker-demo
```
* push镜像到私有库
```sh
docker push reg.codesafe.com/bigbaldy/docker-demo
```
## 获取私有仓库镜像列表
```python
import requests
import json
import traceback

repo_ip = '127.0.0.1'
repo_port = 5000

def getImagesNames(repo_ip,repo_port):
    docker_images = []
    try:
        url = "http://" + repo_ip + ":" +str(repo_port) + "/v2/_catalog"
        res =requests.get(url).content.strip()
        res_dic = json.loads(res)
        images_type = res_dic['repositories']
        for i in images_type:
            url2 = "http://" + repo_ip + ":" +str(repo_port) +"/v2/" + str(i) + "/tags/list"
            res2 =requests.get(url2).content.strip()
            res_dic2 = json.loads(res2)
            name = res_dic2['name']
            tags = res_dic2['tags']
            for tag in tags:
                docker_name = str(repo_ip) + ":" + str(repo_port) + "/" + name + ":" + tag
                docker_images.append(docker_name)
                print docker_name
    except:
        traceback.print_exc()
    return docker_images

print getImagesNames(repo_ip, repo_port)
```
执行就可以看到[u'127.0.0.1:5000/bigbaldy/docker-demo:latest']
## Docker-Compose（一条命令启动多个镜像）
* 安装docker-compose(https://github.com/docker/compose/releases)
* 配置启动文件，例如docker-demo.yml
```yaml
version: '3'
services:
  docker-demo:
    image: bigbaldy/docker-demo
    restart: always
    ports:
      - 8888:8888
```
* 使用docker-compose启动镜像
```sh
docker-compose -f docker-demo.yml up
```
## Maven插件构建Docker镜像
http://blog.csdn.net/qq_22841811/article/details/67369530 //运行会报各种错误！！文章后面有各种解决方式，可以尝试，个人倾向于手动写Dockerfile，然后通过jenkins调用docker命令进行镜像push
## 常用命令
* docker images - 查看镜像列表
* docker ps -a 查看容器列表
* docker run -d -p 8888:8888 reg.codesafe.com/bigbaldy/docker-demo - 启动镜像
* docker start [CONTAINER ID] - 启动容器
* docker stop [CONTAINER ID] - 停止正在运行的容器
* docker rm [CONTAINER ID] - 删除容器，注意必须停止才能删除
* docker rmi [IMAGE ID] - 删除镜像
* docker pull reg.codesafe.com/bigbaldy/docker-demo - 拉取镜像
# Kubernetes安装使用
## minikube
* [官网](https://kubernetes.io/docs/tasks/tools/install-minikube/README.md)
### 1. 安装virtualbox
* yum install SDL
* yum install kernel-devel //注意版本要与系统内核版本一致，若不一致请下载相应的kernel-devel安装包或者升级系统内核（http://blog.csdn.net/u010250863/article/details/70169985）
* 如果安装过KVM，请停止服务
    - systemctl stop libvirtd
    - ps -ef|grep kvm|grep -v grep|cut -c 9-15|xargs kill -9
    - modprobe -r kvm_intel
    - modprobe -r kvm
* yum install gcc
* [下载virtualbox](https://www.virtualbox.org/wiki/Linux_Downloads)
### 2. 安装kubectl 
* 查看最新稳定版：https://storage.googleapis.com/kubernetes-release/release/stable.txt
* 下载，修改版本号即可下载相应版本，例如：https://storage.googleapis.com/kubernetes-release/release/v1.8.3/bin/linux/amd64/kubectl
* chmod +x ./kubectl
* mv ./kubectl /usr/local/bin/kubectl
* [官网](https://kubernetes.io/docs/tasks/tools/install-kubectl/README.md)
### 3. 安装minikube
* [下载](https://github.com/kubernetes/minikube/releases)
### 4. 启动minikube
* minikube start //注意启动过程中会下载localkube，需要翻墙(export http_proxy=http://10.16.13.18:8080)
### 5. 一些前提操作
* minikube ssh
* sudo vim /etc/hosts 添加172.24.62.181 reg.codesafe.com
* scp root@172.24.62.181:/etc/docker/certs.d/reg.codesafe.com/reg.codesafe.com.crt /etc/docker/certs.d/reg.codesafe.com/
* 防止从google拉取镜像
    * docker pull registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0
    * docker tag registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0
### 6. 运行container
```sh
kubectl run docker-demo --image=reg.codesafe.com/bigbaldy/docker-demo --port=8888
```
### 7. 创建服务
```sh
kubectl expose deployment docker-demo --type=LoadBalancer
```
### 8. 打开浏览器访问你的app
```sh
minikube service docker-demo
```
## Web UI (Dashboard)
* [官网](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
* 部署Dashboard UI
```sh
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
* 访问Dashboard UI （未实现，远程机器权限问题还在研究）
## 常用命令
* kubectl get pods - 查看pod
* kubectl describe pod [PodName] - 查看出错原因
* kubectl delete service docker-demo - 删除服务
* kubectl delete deployment docker-demo - 删除部署
* kubectl get services - 查看服务
* kubectl get doployments - 查看部署