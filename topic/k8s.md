
kubeadm: 引导启动k8s集群的命令工具。
kubelet: 在群集中的所有计算机上运行的组件, 并用来执行如启动pods和containers等操作。
kubectl: 用于操作运行中的集群的命令行工具。

#### 安装 docker：
```
yum install -y yum-utils
yum-config-manager     --add-repo https://download.daocloud.io/docker/linux/centos/docker-ce.repo
yum install -y -q --setopt=obsoletes=0 docker-ce-17.03.2.ce* docker-ce-selinux-17.03.2.ce*
systemctl enable docker && systemctl start docker
```
#### SELinux与Firewall
虽然你在上一步已经安装了docker-selinux，但是为了避免出现不必要的麻烦，还是关闭SE为好。
```bash
systemctl stop firewalld && systemctl disable firewalld 
setenforce 0 && sed -i 's/enforcing/disabled/' /etc/sysconfig/selinux
#使用iptables替换掉自带的firewalld
yum install -y iptables-services && systemctl start iptables.service && systemctl enable iptables.service`
```
#### 安装kubelet，kubectl，kubeadm，kubernetes-cni
从源中下载kubeadm，以及依赖包，准备repo：
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
安装组件：
```bash
yum -y install epel-release
yum clean all
yum makecache
yum -y install kubelet kubeadm kubectl kubernetes-cni

systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
```
#### 拉取镜像
```bash
#!/bin/bash
images=(kube-proxy-amd64:v1.11.1 kube-scheduler-amd64:v1.11.1 kube-controller-manager-amd64:v1.11.1 kube-apiserver-amd64:v1.11.1
etcd-amd64:3.1.12 pause-amd64:3.1 kubernetes-dashboard-amd64:v1.8.3 k8s-dns-sidecar-amd64:1.14.10 k8s-dns-kube-dns-amd64:1.14.10
k8s-dns-dnsmasq-nanny-amd64:1.14.10)
for imageName in ${images[@]} ; do
  docker pull dolphintwo/$imageName
  docker tag dolphintwo/$imageName k8s.gcr.io/$imageName
  docker rmi dolphintwo/$imageName
done
```
#### kubeadm安装k8s



https://blog.csdn.net/dt569905467/article/details/81389508
