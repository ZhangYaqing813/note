# CNI_flannel

* [获取安装包](https://github.com/coreos/flannel/releases/download/v0.13.0/flannel-v0.13.0-linux-amd64.tar.gz)
* 解压，获取证书

  * 解压 

    ```
    mkdir /opt/flannel
    tar -zxvf flannel-v0.13.0-linux-amd64.tar.gz -C /opt/flannel
    ```

  * 获取client 证书

    flannel 需要对ETCD数据库进行操作，需要client 端的证书

    ```
    mkdir /opt/flannel/cert
    cd /opt/flannel/cert
    
    scp 10.4.7.200:/opt/certs/ca.pem ./
    scp 10.4.7.200:/opt/certs/.pem ./
    scp 10.4.7.200:/opt/certs/client-key.pem ./
    
    ```

* 创建配置文件

  * 创建 subnet.env

    注意节点IP（FLANNEL_SUBNET=节点IP）

    ```
    vim /opt/flannel/subnet.env
    
    FLANNEL_NETWORK=172.7.0.0/16
    FLANNEL_SUBNET=172.7.21.1/24
    FLANNEL_MTU=1500
    FLANNEL_IPMASQ=false
    ```

  * 创建启动脚本

    public IP 需要修改。

    ```
    vim /opt/flannel/flanneld.sh
    
    #!/bin/sh
    ./flanneld \
    	--public-ip=10.4.7.21\
    	--etcd-endpoints=https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379,
    https://10.4.7.12:2379 \
    	--etcd-keyfile=./cert/client-key.pem
    	--etcd-certfile=./cert/client.pem
    	--etcd-cafile=./cert/ca.pem
    	--iface=ens33
    	--subnet-file=./subnet.env
    	--healthz-port=2041
    ```

  * 添加权限、创建目录

    ```
    chmod +x /opt/flannel/flanned.sh
    mkdir -p /data/logs/flanneld
    
    ```

  * ETCD 配置库添加网络信息

    ```
    cd /opt/etcd
    [root@hdss7-21 etcd]# ./etcdctl set /coreos.com/network/config '{"Network": "172.7.0.0/16","Backend": {"Type": "host-gw"}}'
    
    {"Network": "172.7.0.0/16","Backend": {"Type": "host-gw"}}
    
    验证：
    	./etcdctl get /coreos.com/network/config
    ```

  * 创建supervisor 启动文件

    vim /etc/supervisord.d/flanneld.ini

    ```
    [program:flanneld-7-21]
    command=/opt/flannel/flanneld.sh                                     ; the program (relative uses PATH, can take args)
    numprocs=1                                                           ; number of processes copies to start (def 1)
    directory=/opt/flannel                                               ; directory to cwd to before exec (def no cwd)
    autostart=true                                                       ; start at supervisord start (default: true)
    autorestart=true                                                     ; retstart at unexpected quit (default: true)
    startsecs=30                                                         ; number of secs prog must stay running (def. 1)
    startretries=3                                                       ; max # of serial start failures (default 3)
    exitcodes=0,2                                                        ; 'expected' exit codes for process (default 0,2)
    stopsignal=QUIT                                                      ; signal used to kill process (default TERM)
    stopwaitsecs=10                                                      ; max num secs to wait b4 SIGKILL (default 10)
    user=root                                                            ; setuid to this UNIX account to run the program
    redirect_stderr=true                                                 ; redirect proc stderr to stdout (default false)
    stdout_logfile=/data/logs/flanneld/flanneld.stderr.log               ; stderr log path, NONE for none; default AUTO
    stdout_logfile_maxbytes=64MB                                         ; max # logfile bytes b4 rotation (default 50MB)
    stdout_logfile_backups=4                                             ; # of stdout logfile backups (default 10)
    stdout_capture_maxbytes=1MB                                          ; number of bytes in 'capturemode' (default 0)
    stdout_events_enabled=false 
    ```



  * SNAT 优化(以HDSS7-22 为例)

    >测试现象： 
    >
    >​	启动两个NGINX pod,其中一个对另个一个进行请求时，查看日志发现请求的源IP是宿主机的物理网卡IP，这样无法确定的判断出访问者是谁。

* 安装IPtable-services

  ```
  yum install iptables-services -y
  ```

* 查看iptable 规则

  ```
  [root@hdss7-22 flannel]# iptables-save | grep -i postrouting
  :POSTROUTING ACCEPT [2:120]
  :KUBE-POSTROUTING - [0:0]
  -A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
  -A POSTROUTING -s 172.7.22.0/24 ! -o docker0 -j MASQUERADE
  -A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
  -A KUBE-POSTROUTING -m comment --comment "Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose" -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE
  [root@hdss7-22 flannel]# 
  
  ```

  *  先删除 

    ```
    -A POSTROUTING -s 172.7.22.0/24 ! -o docker0 -j MASQUERADE
    删除：
    iptables -t nat -D POSTROUTING -s 172.7.22.0/24 ! -o docker0 -j MASQUERADE
    
    ```

  * 添加规则

    ```
    iptables -t nat -I POSTROUTING -s 172.7.22.0/24 ! -d 172.7.0.0/16 ! -o docker0 -j MASQUERADE
    ```

  * 保存规则

    ```
    iptables-save > /etc/sysconfig/iptables
    ```

**注意:**

​	修改完成iptables 规则之后如果不能互通，则查看 iptables reject 

