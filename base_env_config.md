# base_env_config
## DNS 安装

* 安装epel 源和常用软件 （所有服务器上）

  ```
  hosnamectl set-hostname hostname #修改主机名
  yum install epel-release -y #安装epel 源
  yum install wget net-tools telnet tree nmap sysstat lrzsz dos2unix bind-utils -y
  
  ```

* 安装基础服务DNS，整个K8S集群采用二进制安装，所以部分基础服务也有手动安装。这里使用的是bind 基础软件

  ```
  host:hdss7-11 
  	 yum install bind -y
  
  ```

  修改配置文件

  ```
  mv /etc/named.conf /etc/named.conf.bak #备份源文件
  vi /etc/named.conf
  需要修改配置
  option{
  	listen-on port 53 { 10.4.7.11; };
  	allow-query     { any; };
  	forwarders      { 10.4.7.254; };
  	dnssec-enable no;
  	dnssec-validation no;
  }；
  
  ```

  编辑配置文件 /etc/named.rfc1912.zones

  ```
  mv /etc/named.rfc1912.zones vim /etc/named.rfc1912.zones
  /etc/named.rfc1912.zones.bak
  添加如下配置
  zone "host.com" IN {
          type  master;
          file  "host.com.zone";
          allow-update { 10.4.7.11; };
  };
  
  zone "od.com" IN {
          type  master;
          file  "od.com.zone";
          allow-update { 10.4.7.11; };
  };
  
  ```

> 编辑配置文件  /var/named/host.com.zone, host.come.zone 
> 是主机名的域，与业务域单独分离开来，互不影响。
> vim /var/named/host.com.zone
  

    ```
      $ORIGIN host.com.
     $TTL 600	; 10 minutes
      @       IN SOA	dns.host.com. dnsadmin.host.com. (
      				2019111001 ; serial
      				10800      ; refresh (3 hours)
      				900        ; retry (15 minutes)
      				604800     ; expire (1 week)
      				86400      ; minimum (1 day)
      				)
      			NS   dns.host.com.
      $TTL 60	; 1 minute
      dns                A    10.4.7.11
      HDSS7-11           A    10.4.7.11
      HDSS7-12           A    10.4.7.12
      HDSS7-21           A    10.4.7.21
      HDSS7-22           A    10.4.7.22
      HDSS7-200          A    10.4.7.200
    ```


  编辑配置文件 /var/named/od.com.zone

  ```
  vim /var/named/od.com.zone
  $ORIGIN od.com.
  $TTL 600	; 10 minutes
  @   		IN SOA	dns.od.com. dnsadmin.od.com. (
  				2019111001 ; serial
  				10800      ; refresh (3 hours)
  				900        ; retry (15 minutes)
  				604800     ; expire (1 week)
  				86400      ; minimum (1 day)
  				)
  				NS   dns.od.com.
  $TTL 60	; 1 minute、
  dns                A    10.4.7.11
  
  ```

  配置完成后修改 /etc/resolv.conf

  ```
  修改nameserver
  	nameserver 10.4.7.11
  	search host.com
  ```

  

## 准备签证书的软件

* hdss7-200 下载证书签发软件cfssl 

  ```
  HDSS7-200上:
  ~]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/bin/cfssl
  ~]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/bin/cfssl-json
  ~]# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/bin/cfssl-certinfo
  ~]# chmod +x /usr/bin/cfssl*
  ```

* 创建根证书csr.json 文件

  ```
  1、创建目录
  	makdir /opt/certs
  2、创建ca-csr.json 文件
  	vim /opt/certs/ca-csr.json
  	
  	/opt/certs/ca-csr.json
  {
      "CN": "OldboyEdu",
      "hosts": [
      ],
      "key": {
          "algo": "rsa",
          "size": 2048
      },
      "names": [
          {
              "C": "CN",
              "ST": "beijing",
              "L": "beijing",
              "O": "od",
              "OU": "ops"
          }
      ],
      "ca": {
          "expiry": "175200h"
      }
  }
  
  
  ```

* 签发根证书

  ```
  cfssl gencert -initca ca-csr.json | cfssl-json -bare ca
  
  ll ./
  -rw-r--r--. 1 root root  993 10月 14 05:48 ca.csr
  -rw-r--r--. 1 root root  329 10月 14 05:48 ca-csr.json
  -rw-------. 1 root root 1675 10月 14 05:48 ca-key.pem
  -rw-r--r--. 1 root root 1346 10月 14 05:48 ca.pem
  
  ```

  

## 安装docker镜像仓库 docker-hub

* 安装docker

  ```
  curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
  ```

* 修改docker配置文件 /etc/docker/daemon.json

  ```
  {
    "graph": "/data/docker",
    "storage-driver": "overlay2",
    "insecure-registries": ["registry.access.redhat.com","quay.io","harbor.od.com"],
    "registry-mirrors": ["https://q2gr04ke.mirror.aliyuncs.com"],
    "bip": "172.7.200.1/24",
    "exec-opts": ["native.cgroupdriver=systemd"],
    "live-restore": true
  }
  ```

  

* 启动docker并加入开机自启

  ```
  systemctl start docker
  systemctl enable docker
  ```

  

* 下载安装文件

  ```
  https://github.com/goharbor/harbor
  #解压到/opt
  tar xf harbor-offline-installer-v1.8.3.tgz -C /opt/  #
  #做个软连接，方便以后升级
  mv harbor/ harbor-v1.8.3
  ln -s /opt/harbor-v1.8.3/ /opt/harbor
  ```

* 修改配置文件  /opt/harbor/harbor.yml 创建相关目录

  ```
  cp harbor.yml.tmpl harbor.yml
  vim /opt/harbor/harbor.yml
  修改：
  hostname: harbor.od.com
  http:
    port: 180
  #数据存放目录
  data_volume: /data/harbor
  #日志存放目录
  location: /data/harbor/logs
  #创建目录
  mkdir -p /data/harbor/logs
  
  ```

* 安装docker-compos 工具，harbor 的安装依赖单机编排工具 docker-compose,执行harbor 安装脚本

  ```
  #安装docker-compse
  yum install docker-compose -y
  #执行脚本
  sh /opt/harbor/install.sh
  #查看执行结果
  docker-compose ps
  ```

  

* 安装NGINX ，修改配置文件，代理harbor 站点

  ```
  #安装NGINX
  yum install nginx -y
  #修改配置文件
  vim /etc/nginx/conf.d/harbor.od.com.conf
  server {
      listen       80;
      server_name  harbor.od.com;
  
      client_max_body_size 1000m;
  
      location / {
          proxy_pass http://127.0.0.1:180;
      }
  }
  #启动NGINX
  nginx -t  检测配置文件
  systemctl start nginx && systemctl enable nginx 
  
  ```

  

* 添加DNS 解析 HDSS7-11

  ```
  #DNS配置文件
  vim /var/named/od.com.zone
  添加：
  harbor             A    10.4.7.200
  
  ```
**注意：**
   > 序号前滚+1
  

* 测试

  ```
  DNS 添加完成后进行简单的测试
  修改宿主机上的DNS指向 10.4.7.11
  
  #浏览器打开http://harbor.od.com
  
  用户名：admin
  密码：Harbor12345
  
  #docker 拉取镜像
  	docker pull nginx:1.7.9
  #给新拉取的镜像打个新的tag
  	docker tag 84581e99d807 harbor.od.com/public/nginx:v1.7.9
  #登录  harbor.od.com
  	docker login harbor.od.com
  	[root@hdss7-200 harbor]# docker login harbor.od.com
  	Username: admin
  	Password: 
  	WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
  	Configure a credential helper to remove this warning. See
  	https://docs.docker.com/engine/reference/commandline/login/#credentials-store
  
  	Login Succeeded
  	登录成功
  	
  #把新打好的tag nginx 镜像上传到镜像仓库
  	docker push harbor.od.com/public/nginx:v1.7.9
  
  
  ```

  

* 安装 supervisor 

  ```
  #安装
  yum install supervisor -y
  #启动并加入开机自启
  systemctl start supervisord 
  systemctl enable supervisord
  ```