# Core_Plugin_Install
# Kubernetes API Server 安装

* 获取安装包

  ```
  https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.15.md#downloads-for-v1152
  ```

* 解压

  ```
  cd /opt
  mv kubernetes kubernetes-1.15.2
  # 做软链
  ln -s kubernetes-1.15.2 kubernetes
  
  ```

* 删除一些不必要的包

  ```
  cd kubernetes/server/bin
  rm -rf *.tar
  rm -rf *_tag
  
  [root@hdss7-21 bin]# ll
  total 884636
  -rwxr-xr-x. 1 root root  43534816 8月   5 2019 apiextensions-apiserver
  -rwxr-xr-x. 1 root root 100548640 8月   5 2019 cloud-controller-manager
  -rwxr-xr-x. 1 root root 200648416 8月   5 2019 hyperkube
  -rwxr-xr-x. 1 root root  40182208 8月   5 2019 kubeadm
  -rwxr-xr-x. 1 root root 164501920 8月   5 2019 kube-apiserver
  -rwxr-xr-x. 1 root root 116397088 8月   5 2019 kube-controller-manager
  -rwxr-xr-x. 1 root root  42985504 8月   5 2019 kubectl
  -rwxr-xr-x. 1 root root 119616640 8月   5 2019 kubelet
  -rwxr-xr-x. 1 root root  36987488 8月   5 2019 kube-proxy
  -rwxr-xr-x. 1 root root  38786144 8月   5 2019 kube-scheduler
  -rwxr-xr-x. 1 root root   1648224 8月   5 2019 mounter
  [root@hdss7-21 bin]# 
  
  ```

* 证书准备与签发

  > 1、client端证书配置文件 client-csr.json

  ```
  {
      "CN": "k8s-node",
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
      ]
  }
  ```

  > 2、server端证书配置apiserver-csr.json

  ```
  {
      "CN": "k8s-apiserver",
      "hosts": [
          "127.0.0.1",
          "192.168.0.1",
          "kubernetes.default",
          "kubernetes.default.svc",
          "kubernetes.default.svc.cluster",
          "kubernetes.default.svc.cluster.local",
          "10.4.7.10",
          "10.4.7.21",
          "10.4.7.22",
          "10.4.7.23"
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
      ]
  }
  ```

  

  > 3、签发证书

  ```
  # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client-csr.json |cfssl-json -bare client
  
  # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server apiserver-csr.json |cfssl-json -bare apiserver
  ```

  

* 分发证书

  > 1、分发证书到hdss7-21|22

  ```
  cd /opt/kubernetes/server/bin/cert
  
  scp 10.4.7.200:/opt/certs/apiserver.pem ./
  scp 10.4.7.200:/opt/certs/apiserver-key.pem ./
  scp 10.4.7.200:/opt/certs/client-key.pem ./
  scp 10.4.7.200:/opt/certs/client.pem ./
  scp 10.4.7.200:/opt/certs/ca-key.pem ./
  scp 10.4.7.200:/opt/certs/ca.pem ./
  
  ```

  > 2、修改证书权限

  ```
  chmod 600 ./*.pem
  ll
  
  -rw-------. 1 root root 1675 10月 15 05:12 apiserver-key.pem
  -rw-------. 1 root root 1598 10月 15 05:12 apiserver.pem
  -rw-------. 1 root root 1675 10月 15 05:13 client-key.pem
  -rw-------. 1 root root 1363 10月 15 05:13 client.pem
  -rw-------. 1 root root 1675 10月 15 05:17 ca-key.pem
  -rw-------. 1 root root 1346 10月 15 05:17 ca.pem
  
  ```

  

* 创建apiserver 配置文件

  ```
  mkdir conf
  vim audit.yaml
  
  apiVersion: audit.k8s.io/v1beta1 # This is required.
  kind: Policy
  # Don't generate audit events for all requests in RequestReceived stage.
  omitStages:
    - "RequestReceived"
  rules:
    # Log pod changes at RequestResponse level
    - level: RequestResponse
      resources:
      - group: ""
        # Resource "pods" doesn't match requests to any subresource of pods,
        # which is consistent with the RBAC policy.
        resources: ["pods"]
    # Log "pods/log", "pods/status" at Metadata level
    - level: Metadata
      resources:
      - group: ""
        resources: ["pods/log", "pods/status"]
  
    # Don't log requests to a configmap called "controller-leader"
    - level: None
      resources:
      - group: ""
        resources: ["configmaps"]
        resourceNames: ["controller-leader"]
  
    # Don't log watch requests by the "system:kube-proxy" on endpoints or services
    - level: None
      users: ["system:kube-proxy"]
      verbs: ["watch"]
      resources:
      - group: "" # core API group
        resources: ["endpoints", "services"]
  
    # Don't log authenticated requests to certain non-resource URL paths.
    - level: None
      userGroups: ["system:authenticated"]
      nonResourceURLs:
      - "/api*" # Wildcard matching.
      - "/version"
  
    # Log the request body of configmap changes in kube-system.
    - level: Request
      resources:
      - group: "" # core API group
        resources: ["configmaps"]
      # This rule only applies to resources in the "kube-system" namespace.
      # The empty string "" can be used to select non-namespaced resources.
      namespaces: ["kube-system"]
  
    # Log configmap and secret changes in all other namespaces at the Metadata level.
    - level: Metadata
      resources:
      - group: "" # core API group
        resources: ["secrets", "configmaps"]
  
    # Log all other resources in core and extensions at the Request level.
    - level: Request
      resources:
      - group: "" # core API group
      - group: "extensions" # Version of group should NOT be included.
  
    # A catch-all rule to log all other requests at the Metadata level.
    - level: Metadata
      # Long-running requests like watches that fall under this rule will not
      # generate an audit event in RequestReceived.
      omitStages:
        - "RequestReceived"
  
  ```

  

* 创建 apiserver 启动脚本

  > 1、vim  /opt/kubernetes/server/bin/kube-apiserver.sh && chmod +x /opt/kubernetes/server/bin/kube-apiserver.sh 

  ```
  #!/bin/bash
  ./kube-apiserver \
    --apiserver-count 2 \
    --audit-log-path /data/logs/kubernetes/kube-apiserver/audit-log \
    --audit-policy-file ./conf/audit.yaml \
    --authorization-mode RBAC \
    --client-ca-file ./cert/ca.pem \
    --requestheader-client-ca-file ./cert/ca.pem \
    --enable-admission-plugins NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
    --etcd-cafile ./cert/ca.pem \
    --etcd-certfile ./cert/client.pem \
    --etcd-keyfile ./cert/client-key.pem \
    --etcd-servers https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379 \
    --service-account-key-file ./cert/ca-key.pem \
    --service-cluster-ip-range 192.168.0.0/16 \
    --service-node-port-range 3000-29999 \
    --target-ram-mb=1024 \
    --kubelet-client-certificate ./cert/client.pem \
    --kubelet-client-key ./cert/client-key.pem \
    --log-dir  /data/logs/kubernetes/kube-apiserver \
    --tls-cert-file ./cert/apiserver.pem \
    --tls-private-key-file ./cert/apiserver-key.pem \
    --v 2
  
  ```

  > 2、创建相关目录

   ```
  mkdir -p /data/logs/kubernetes/kube-apiserver
   ```

  > 3、创建supervisor 的配置文件   vi /etc/supervisord.d/kube-apiserver.ini

  ```
  [program:kube-apiserver-7-21]
  command=/opt/kubernetes/server/bin/kube-apiserver.sh            ; the program (relative uses PATH, can take args)
  numprocs=1                                                      ; number of processes copies to start (def 1)
  directory=/opt/kubernetes/server/bin                            ; directory to cwd to before exec (def no cwd)
  autostart=true                                                  ; start at supervisord start (default: true)
  autorestart=true                                                ; retstart at unexpected quit (default: true)
  startsecs=30                                                    ; number of secs prog must stay running (def. 1)
  startretries=3                                                  ; max # of serial start failures (default 3)
  exitcodes=0,2                                                   ; 'expected' exit codes for process (default 0,2)
  stopsignal=QUIT                                                 ; signal used to kill process (default TERM)
  stopwaitsecs=10                                                 ; max num secs to wait b4 SIGKILL (default 10)
  user=root                                                       ; setuid to this UNIX account to run the program
  redirect_stderr=true                                            ; redirect proc stderr to stdout (default false)
  stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stdout.log        ; stderr log path, NONE for none; default AUTO
  stdout_logfile_maxbytes=64MB                                    ; max # logfile bytes b4 rotation (default 50MB)
  stdout_logfile_backups=4                                        ; # of stdout logfile backups (default 10)
  stdout_capture_maxbytes=1MB                                     ; number of bytes in 'capturemode' (default 0)
  stdout_events_enabled=false                                     ; emit events on stdout writes (default false)
  ```

  

*  安装 NGINX 代理  (hdss7-11,hdss7-12)

> 1、安装NGINX

```
yum install nginx -y
```

> 2、配置NGINX  vim /etc/nginx/nginx.conf

```
stream {
    upstream kube-apiserver {
        server 10.4.7.21:6443     max_fails=3 fail_timeout=30s;
        server 10.4.7.22:6443     max_fails=3 fail_timeout=30s;
    }
    server {
        listen 7443;
        proxy_connect_timeout 2s;
        proxy_timeout 900s;
        proxy_pass kube-apiserver;
    }
}
```

> 3、启动NGINX 并加入开机启动

```
systemctl start nginx 
systemctl enable nginx
```



* 安装keepalived (hdss7-11,hdss7-12)

> 1、安装keepalived

```
yum install keepalived -y
```

> 2、创建检测脚本 vim /etc/keepalived/check_port.sh

```

#!/bin/bash
#keepalived 监控端口脚本
#使用方法：
#在keepalived的配置文件中
#vrrp_script check_port {#创建一个vrrp_script脚本,检查配置
#    script "/etc/keepalived/check_port.sh 6379" #配置监听的端口
#    interval 2 #检查脚本的频率,单位（秒）
#}
CHK_PORT=$1
if [ -n "$CHK_PORT" ];then
        PORT_PROCESS=`ss -lnt|grep $CHK_PORT|wc -l`
        if [ $PORT_PROCESS -eq 0 ];then
                echo "Port $CHK_PORT Is Not Used,End."
                exit 1
        fi
else
        echo "Check Port Cant Be Empty!"
fi

赋权 
	chmod +x /etc/keepalived/check_port.sh
```

> 3、 修改配置文件 /etc/keepalived/keepalived.conf

```
keepalived 主:

! Configuration File for keepalived

global_defs {
   router_id 10.4.7.11
   script_user root
   enable_script_security 

}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 7443"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 251
    priority 100
    advert_int 1
    mcast_src_ip 10.4.7.11
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    track_script {
         chk_nginx
    }
    virtual_ipaddress {
        10.4.7.10
    }
}

keepalived从:

! Configuration File for keepalived
global_defs {
	router_id 10.4.7.12
	script_user root
    enable_script_security 
}
vrrp_script chk_nginx {
	script "/etc/keepalived/check_port.sh 7443"
	interval 2
	weight -20
}
vrrp_instance VI_1 {
	state BACKUP
	interface eth0
	virtual_router_id 251
	mcast_src_ip 10.4.7.12
	priority 90
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 11111111
	}
	track_script {
		chk_nginx
	}
	virtual_ipaddress {
		10.4.7.10
	}
}
```

> 4、启动keepalived 

```
systemctl start keepalived
systemctl enable keepalived

```

# K8S kube-controller manager

* 创建启动脚本 cube-controller-manager.sh 

```
vim /opt/kubernetes/server/bin/kube-controller-manager.sh

#!/bin/bash
./kube-controller-manager \
  --cluster-cidr 172.7.0.0/16 \
  --leader-elect true \
  --log-dir /data/logs/kubernetes/kube-controller-manager \
  --master http://127.0.0.1:8080 \
  --service-account-private-key-file ./cert/ca-key.pem \
  --service-cluster-ip-range 192.168.0.0/16 \
  --root-ca-file ./cert/ca.pem \
  --v 2
```

* 修改权限并创建日志目录

```
chmod +x  /opt/kubernetes/server/bin/kube-controller-manager.sh
mkdir -p /data/logs/kubernetes/kube-controller-manager

```

* 创建supervisor 配置文件

```
 vim /etc/supervisord.d/kube-conntroller-manager.ini
 
 [program:kube-controller-manager-7-21]
command=/opt/kubernetes/server/bin/kube-controller-manager.sh                     ; the program (relative uses PATH, can take args)
numprocs=1                                                                        ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                                              ; directory to cwd to before exec (def no cwd)
autostart=true                                                                    ; start at supervisord start (default: true)
autorestart=true                                                                  ; retstart at unexpected quit (default: true)
startsecs=30                                                                      ; number of secs prog must stay running (def. 1)
startretries=3                                                                    ; max # of serial start failures (default 3)
exitcodes=0,2                                                                     ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                                   ; signal used to kill process (default TERM)
stopwaitsecs=10                                                                   ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                                         ; setuid to this UNIX account to run the program
redirect_stderr=true                                                              ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controller.stdout.log  ; stderr log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                                      ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                                          ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                                       ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                                       ; emit events on stdout writes (default false)
 
```

# kube-scheduler



* 创建启动脚本

```
vim opt/kubernetes/server/bin/kube-scheduler.sh

#!/bin/sh
./kube-scheduler \
  --leader-elect  \
  --log-dir /data/logs/kubernetes/kube-scheduler \
  --master http://127.0.0.1:8080 \
  --v 2
```

* 修改权限&&创建日志目录

```
chmod +x /opt/kubernetes/server/bin/kube-scheduler.sh
mkdir -p /data/logs/kubernetes/kube-scheduler
```



* 创建supervisor 配置文件

```
vim /etc/supervisord.d/kube-scheduler.ini

[program:kube-scheduler-7-21]
command=/opt/kubernetes/server/bin/kube-scheduler.sh                     ; the program (relative uses PATH, can take args)
numprocs=1                                                               ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                                     ; directory to cwd to before exec (def no cwd)
autostart=true                                                           ; start at supervisord start (default: true)
autorestart=true                                                         ; retstart at unexpected quit (default: true)
startsecs=30                                                             ; number of secs prog must stay running (def. 1)
startretries=3                                                           ; max # of serial start failures (default 3)
exitcodes=0,2                                                            ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                          ; signal used to kill process (default TERM)
stopwaitsecs=10                                                          ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                                ; setuid to this UNIX account to run the program
redirect_stderr=true                                                     ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stdout.log ; stderr log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                             ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                                 ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                              ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                              ; emit events on stdout writes (default false)
```



# kubectl 

* 创建kubectl 软连接文件

  ```
  ln -s /opt/kubernetes/server/bin/kubectl /usr/bin/kubectl
  
  [root@hdss7-21 bin]# kubectl get cs
  NAME                 STATUS    MESSAGE              ERROR
  scheduler            Healthy   ok                   
  controller-manager   Healthy   ok                   
  etcd-1               Healthy   {"health": "true"}   
  etcd-2               Healthy   {"health": "true"}   
  etcd-0               Healthy   {"health": "true"}   
  ```

  

