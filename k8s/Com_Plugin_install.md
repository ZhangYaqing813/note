# Com_Plugin_install
# K8S-node 节点部署

> 1、签发证书

* 创建证书配置文件 (hdss7-200)

  vim vi kubelet-csr.json

  ```
  {
      "CN": "k8s-kubelet",
      "hosts": [
      "127.0.0.1",
      "10.4.7.10",
      "10.4.7.21",
      "10.4.7.22",
      "10.4.7.23",
      "10.4.7.24",
      "10.4.7.25",
      "10.4.7.26",
      "10.4.7.27",
      "10.4.7.28"
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

* 生成证书

  ```
  # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server kubelet-csr.json | cfssl-json -bare kubelet
  
  ```

  

* 分发证书 

  ```
  root@hdss7-21 cert]# scp 10.4.7.200:/opt/certs/kubelet.pem /opt/kubernetes/server/bin/cert
  root@hdss7-21 cert]# scp 10.4.7.200:/opt/certs/kubelet-key.pem /opt/kubernetes/server/bin/cert
  
  ```

> 2、生成配置文件

* 配置 set-cluster 

  ```
  conf]# kubectl config set-cluster myk8s \
    --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
    --embed-certs=true \
    --server=https://10.4.7.10:7443 \
    --kubeconfig=kubelet.kubeconfig
    
  Cluster "myk8s" set.
  
  ```

* 配置 set-credentials

  ```
  conf]# kubectl config set-credentials k8s-node \
    --client-certificate=/opt/kubernetes/server/bin/cert/client.pem \
    --client-key=/opt/kubernetes/server/bin/cert/client-key.pem \
    --embed-certs=true \
    --kubeconfig=kubelet.kubeconfig 
    
  User "k8s-node" set.
  ```

  

* 配置 set-context

  ```
  conf]# kubectl config set-context myk8s-context \
    --cluster=myk8s \
    --user=k8s-node \
    --kubeconfig=kubelet.kubeconfig
    
  Context "myk8s-context" created.
  ```

  

* 配置 use-context

  ```
  conf]# kubectl config use-context myk8s-context --kubeconfig=kubelet.kubeconfig
  
  Switched to context "myk8s-context".
  
  [root@hdss7-21 conf]# ll
  total 12
  -rw-r--r--. 1 root root 2222 10月 15 05:26 audit.yaml
  -rw-------  1 root root 6195 10月 20 04:28 kubelet.kubeconfig
  
  ```

> 3、创建用户资源配置文件

* k8s-node.yaml

  ```
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: k8s-node
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:node
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: k8s-node
  ```

  



* 创建集群用户资源

  ```
  kubectl create -f k8s-node.yaml
  ```

  

>  4、拉取启动需要的镜像文件 puse (HDSS7-200) 

 *  拉取镜像上传到私有仓库

    ```
    docker pull kubernetes/pause
    docker tag f9d5de079539 harbor.od.com/public/pause:latest
    docker push harbor.od.com/public/pause:latest
    ```

   

> 5、创建启动脚本

 * /opt/kubernetes/server/bin/kubelet.sh

   ```
   #!/bin/sh
   ./kubelet \
     --anonymous-auth=false \
     --cgroup-driver systemd \
     --cluster-dns 192.168.0.2 \
     --cluster-domain cluster.local \
     --runtime-cgroups=/systemd/system.slice \
     --kubelet-cgroups=/systemd/system.slice \
     --fail-swap-on="false" \
     --client-ca-file ./cert/ca.pem \
     --tls-cert-file ./cert/kubelet.pem \
     --tls-private-key-file ./cert/kubelet-key.pem \
     --hostname-override hdss7-21.host.com \
     --image-gc-high-threshold 20 \
     --image-gc-low-threshold 10 \
     --kubeconfig ./conf/kubelet.kubeconfig \
     --log-dir /data/logs/kubernetes/kube-kubelet \
     --pod-infra-container-image harbor.od.com/public/pause:latest \
     --root-dir /data/kubelet
   ```

   

* 赋权，创建日志目录

  ```
  # mkdir -p /data/logs/kubernetes/kube-kubelet /data/kubelet
  # chmod +x kubelet.sh
  
  ```

> 6、创建supervisor 配置文件

* vim /etc/supervisord.d/kube-kubelet.ini

  ```
  /etc/supervisord.d/kube-kubelet.ini
  [program:kube-kubelet-7-21]
  command=/opt/kubernetes/server/bin/kubelet.sh     ; the program (relative uses PATH, can take args)
  numprocs=1                                        ; number of processes copies to start (def 1)
  directory=/opt/kubernetes/server/bin              ; directory to cwd to before exec (def no cwd)
  autostart=true                                    ; start at supervisord start (default: true)
  autorestart=true              		          ; retstart at unexpected quit (default: true)
  startsecs=30                                      ; number of secs prog must stay running (def. 1)
  startretries=3                                    ; max # of serial start failures (default 3)
  exitcodes=0,2                                     ; 'expected' exit codes for process (default 0,2)
  stopsignal=QUIT                                   ; signal used to kill process (default TERM)
  stopwaitsecs=10                                   ; max num secs to wait b4 SIGKILL (default 10)
  user=root                                         ; setuid to this UNIX account to run the program
  redirect_stderr=true                              ; redirect proc stderr to stdout (default false)
  stdout_logfile=/data/logs/kubernetes/kube-kubelet/kubelet.stdout.log   ; stderr log path, NONE for none; default AUTO
  stdout_logfile_maxbytes=64MB                      ; max # logfile bytes b4 rotation (default 50MB)
  stdout_logfile_backups=4                          ; # of stdout logfile backups (default 10)
  stdout_capture_maxbytes=1MB                       ; number of bytes in 'capturemode' (default 0)
  stdout_events_enabled=false                       ; emit events on stdout writes (default false)
  
  ```

  

# kube-proxy

> 1、准备工作
>
> 检查ipvs 模块是加载

 * vim ipvs.sh  执行脚本

   ```
   #!/bin/bash
   ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
   for i in $(ls $ipvs_mods_dir|grep -o "^[^.]*")
   do
     /sbin/modinfo -F filename $i &>/dev/null
     if [ $? -eq 0 ];then
       /sbin/modprobe $i
     fi
   done
   ```

> 2、创建启动脚本 

 * vim /opt/kubernetes/server/bin/kube-proxy.sh

   ```
   #!/bin/sh
   ./kube-proxy \
     --cluster-cidr 172.7.0.0/16 \
     --hostname-override hdss7-21.host.com \
     --proxy-mode=ipvs \
     --ipvs-scheduler=nq \
     --kubeconfig ./conf/kube-proxy.kubeconfig
   ```



> 3、生成证书及配置文件

 * 生成证书

    * 创建配置文件 

      vi kube-proxy-csr.json

      ```
      {
          "CN": "system:kube-proxy",
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

   * 生成证书 

     ```
     # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client kube-proxy-csr.json |cfssl-json -bare kube-proxy-client
     ```

   * 分发证书

     ```
      308  scp ./kube-proxy-client.pem 10.4.7.21:/opt/kubernetes/server/bin/cert/
      309  scp ./kube-proxy-client-key.pem 10.4.7.21:/opt/kubernetes/server/bin/cert/
      310  scp ./kube-proxy-client.pem 10.4.7.22:/opt/kubernetes/server/bin/cert/
      311  scp ./kube-proxy-client-key.pem 10.4.7.22:/opt/kubernetes/server/bin/cert/
     
     ```

   * 生成配置文件

     * set-cluster 

       ```
       conf]# kubectl config set-cluster myk8s \
         --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
         --embed-certs=true \
         --server=https://10.4.7.10:7443 \
         --kubeconfig=kube-proxy.kubeconfig
       ```

       

     * set-credentials

       ```
       conf]# kubectl config set-credentials kube-proxy \
         --client-certificate=/opt/kubernetes/server/bin/cert/kube-proxy-client.pem \
         --client-key=/opt/kubernetes/server/bin/cert/kube-proxy-client-key.pem \
         --embed-certs=true \
         --kubeconfig=kube-proxy.kubeconfig
       ```

       

     * set-context

       ```
       conf]# kubectl config set-context myk8s-context \
         --cluster=myk8s \
         --user=kube-proxy \
         --kubeconfig=kube-proxy.kubeconfig
       ```

       

     * use-context

       ```
       conf]# kubectl config use-context myk8s-context --kubeconfig=kube-proxy.kubeconfig
       ```

       

     

 

> 3、 创建supervisor 启动文件

 * vim /etc/supervisord.d/kube-proxy.ini

   ```
   [program:kube-proxy-7-21]
   command=/opt/kubernetes/server/bin/kube-proxy.sh                     ; the program (relative uses PATH, can take args)
   numprocs=1                                                           ; number of processes copies to start (def 1)
   directory=/opt/kubernetes/server/bin                                 ; directory to cwd to before exec (def no cwd)
   autostart=true                                                       ; start at supervisord start (default: true)
   autorestart=true                                                     ; retstart at unexpected quit (default: true)
   startsecs=30                                                         ; number of secs prog must stay running (def. 1)
   startretries=3                                                       ; max # of serial start failures (default 3)
   exitcodes=0,2                                                        ; 'expected' exit codes for process (default 0,2)
   stopsignal=QUIT                                                      ; signal used to kill process (default TERM)
   stopwaitsecs=10                                                      ; max num secs to wait b4 SIGKILL (default 10)
   user=root                                                            ; setuid to this UNIX account to run the program
   redirect_stderr=true                                                 ; redirect proc stderr to stdout (default false)
   stdout_logfile=/data/logs/kubernetes/kube-proxy/proxy.stdout.log     ; stderr log path, NONE for none; default AUTO
   stdout_logfile_maxbytes=64MB                                         ; max # logfile bytes b4 rotation (default 50MB)
   stdout_logfile_backups=4                                             ; # of stdout logfile backups (default 10)
   stdout_capture_maxbytes=1MB                                          ; number of bytes in 'capturemode' (default 0)
   stdout_events_enabled=false                                          ; emit events on stdout writes (default false)
   ```

   

> 4、授权并创建目录

 * 授权

   ```
   chmod +x kube-proxy.sh
   ```

* 创建目录

  ```
  mkdir -p /data/logs/kubernetes/kube-proxy
  ```

  

* 更新supervisor

  ```
  supervisorctl update
  ```

  