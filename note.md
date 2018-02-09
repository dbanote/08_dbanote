
sudo docker pull oraclelinux

sudo docker run --hostname yderp -it oraclelinux /bin/bash


sudo docker pull tomcat:8

sudo docker run --name tomcat-server --hostname tomcat-server --privileged=true -v /app/tomcat8/www:/usr/local/tomcat/webapps/www  -p 8080:8080 tomcat:8 


sudo docker pull centos:7



mkdir baseos
cd baseos
wget http://mirrors.aliyun.com/repo/Centos-7.repo

vi Centos-7.repo
删除包含 http://mirrors.aliyuncs.com的行


vi Dockerfile

FROM centos:7
MAINTAINER liuyajun <liuyajun@ydgw.cn>
LABEL name="CentOS7.4 Base Image" \
      description="update aliyun repo, update system" \
      build-date="20180119"

RUN \cp -rf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
ADD Centos-7.repo /etc/yum.repos.d/

RUN yum makecache
RUN yum -y update

CMD ["/bin/bash"]


sudo docker build -t foxbei/centos7:20180119 .
sudo docker run -it foxbei/centos7:20180119


sudo docker login

sudo docker push foxbei/centos7:20180119



sudo usermod -aG docker ubuntu



sudo docker tag foxbei/centos7:20180119 10.245.231.201:5000/centos7:20180119


docker push 10.245.231.201:5000/centos7:20180119

docker pull 10.245.231.201:5000/centos7:20180119

curl http://10.245.231.201:5000/v2/_catalog
sudo systemctl daemon-reload 
sudo systemctl restart docker


sudo apt install lrzsz

mkdir jre
cd jre


vi Dockerfile

FROM foxbei/centos7:20180119
MAINTAINER liuyajun <liuyajun@ydgw.cn>
LABEL name="jre-8u161-linux-x64" \
      description="jre-8u161-linux-x64" \
      build-date="20180119"

ADD jre-8u161-linux-x64.rpm .
RUN yum -y localinstall jre-8u161-linux-x64.rpm && rm -rf jre-8u161-linux-x64.rpm

ENV JAVA_HOME /usr/java/jre1.8.0_161


docker build -t foxbei/jre1.8:20180119 .

docker run -it foxbei/jre1.8:20180119 /bin/bash



构建Tomcat基础镜像

mkdir tomcat8-jre8
cd tomcat8-jre8

rz
apache-tomcat-8.5.24.tar.gz

vi Dockerfile

FROM foxbei/jre
MAINTAINER liuyajun <liuyajun@ydgw.cn>
LABEL name="tomcat8-jre8" \
      description="This image is used to run tomcat8 with jre8" \
      build-date="20180119"

ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
WORKDIR $CATALINA_HOME

ADD apache-tomcat-8.5.24 .

EXPOSE 8080
CMD ["catalina.sh", "run"]


docker build -t foxbei/tomcat8-jre8:20180119 .

docker run -d --name tomcat8-dev --hostname tomcat8-dev -p 8088:8080 foxbei/tomcat8-jre8:20180119


yum install openssh-server



检查k8s状态
kubectl get pod --namespace=kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
heapster-6f6f5dcf6c-ddmh6               1/1       Running   0          3d
kube-dns-78c55df97c-psjl7               3/3       Running   0          3d
kubernetes-dashboard-68b6bb6c7b-xv58b   1/1       Running   0          3d
monitoring-grafana-5b9f7464b9-nn6bj     1/1       Running   0          3d
monitoring-influxdb-767b96bb5b-m6ksv    1/1       Running   1          3d
tiller-deploy-7dffbf6bd9-s6sq4          1/1       Running   0          3d








tomcat:/usr/local/tomcat/webapps

sudo docker history craig/tomcat



名称  挂载点 快照时间线   操作
/opt/dc-nas/fs/ext4_A/pool_15K-lun_15K/docker/tomcat:/usr/local/tomcat/webapps


mount -t nfs 10.240.3.200:/docker /data -o nolock


sudo echo "docker-ce install" | sudo dpkg --set-selections




docker run -it foxbei/tomcat /bin/bash


echo "122.158.133.98 mirrors.aliyun.com" >> /etc/hosts

yum install -y passwd openssl openssh-server && \
ssh-keygen -q -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key -N '' && \
ssh-keygen -q -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N '' && \
ssh-keygen -t dsa -f /etc/ssh/ssh_host_ed25519_key -N '' && \
sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config && \
sed -i "s/UsePAM.*/UsePAM no/g" /etc/ssh/sshd_config

passwd root

docker ps -all
docker commit 2904984e6fa5 foxbei/tomcat:ssh
docker rm -f 2904984e6fa5
docker run -d -p 50022:22 foxbei/tomcat:ssh /usr/sbin/sshd -D


vi Dockerfile
FROM foxbei/jre
MAINTAINER liuyajun <liuyajun@ydgw.cn>
LABEL name="tomcat-ssh" \
      description="tomcat ssh open" \
      build-date="20180124"

ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
WORKDIR $CATALINA_HOME

ADD apache-tomcat-8.5.24 .

RUN yum install -y passwd openssl openssh-server && \
ssh-keygen -q -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key -N '' && \
ssh-keygen -q -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N '' && \
ssh-keygen -t dsa -f /etc/ssh/ssh_host_ed25519_key -N '' && \
sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config && \
sed -i "s/UsePAM.*/UsePAM no/g" /etc/ssh/sshd_config && \
echo "dev123456+@" | passwd --stdin root

EXPOSE 8080 22
CMD ["catalina.sh", "run"]
CMD ["/usr/sbin/sshd", "-D"]





tom201:/usr/local/tomcat/webapps
rancher-nfs
rancher-nfs



mount -t nfs -o nolock 10.240.3.200:/docker /data

chown nobody:nogroup /data

chmod 777 /data

retain


mount -t nfs -o nolock 10.245.3.200:/docker /data


nfsstat -m


sudo echo "docker-ce install" | sudo dpkg --set-selections


docker run -it oraclelinux /bin/bash


mount -o loop /iso/OEL7.4.iso /media/disk

vi Dockerfile
FROM oraclelinux
MAINTAINER liuyajun <liuyajun@ydgw.cn>
LABEL name="oracle linux 7.4" \
      build-date="20180126"

ADD disk /media/disk
RUN mv /etc/yum.repos.d/public-yum-ol7.repo /etc/yum.repos.d/public-yum-ol7.repo.bak && \
echo "[ol7_latest]" >> /etc/yum.repos.d/public-yum-ol7.repo && \
echo "name=Oracle Linux 7.4" >> /etc/yum.repos.d/public-yum-ol7.repo && \
echo "baseurl=file:///media/disk" >> /etc/yum.repos.d/public-yum-ol7.repo && \
echo "gpgcheck=0" >> /etc/yum.repos.d/public-yum-ol7.repo && \
echo "enabled=1" >> /etc/yum.repos.d/public-yum-ol7.repo && \

docker build -t foxbei/oracle11g .



echo "[ol7_latest]" >> /etc/yum.repos.d/public-yum-ol7.repo
echo "name=Oracle Linux 7.4" >> /etc/yum.repos.d/public-yum-ol7.repo
echo "baseurl=file:///media/disk" >> /etc/yum.repos.d/public-yum-ol7.repo
echo "gpgcheck=0" >> /etc/yum.repos.d/public-yum-ol7.repo
echo "enabled=1" >> /etc/yum.repos.d/public-yum-ol7.repo


docker run -it foxbei/oracle11g /bin/bash
