# ETCD_Install
* ETCD集群通信证书 （证书的签发均在hdss7-200 上进行）

  ```
  1、编辑根证书信息
  vim /opt/certs/ca-config.json
  
  {
      "signing": {
          "default": {
              "expiry": "175200h"
          },
          "profiles": {
              "server": {
                  "expiry": "175200h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth"
                  ]
              },
              "client": {
                  "expiry": "175200h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "client auth"
                  ]
              },
              "peer": {
                  "expiry": "175200h",
                  "usages": [
                      "signing",
                      "key encipherment",
                      "server auth",
                      "client auth"
                  ]
              }
          }
      }
  }
  ```

 *  2、创建etcd 证书文件
 > vim etcd-peer-csr.json
       
```
      {
          "CN": "k8s-etcd",
          "hosts": [
              "10.4.7.11",
              "10.4.7.12",
              "10.4.7.21",
              "10.4.7.22"
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
  
* 3、**生成etcd-peer证书**
    `cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-peer-csr.json |cfssl-json -bare etcd-peer`

```
      [root@hdss7-200 certs]# ll
      total 36
      -rw-r--r--. 1 root root  837 10月 15 14:20 ca-config.json
      -rw-r--r--. 1 root root  993 10月 14 17:48 ca.csr
      -rw-r--r--. 1 root root  329 10月 14 17:48 ca-csr.json
      -rw-------. 1 root root 1675 10月 14 17:48 ca-key.pem
      -rw-r--r--. 1 root root 1346 10月 14 17:48 ca.pem
      -rw-r--r--. 1 root root 1062 10月 15 14:25 etcd-peer.csr
      -rw-r--r--. 1 root root  363 10月 15 14:21 etcd-peer-csr.json
      -rw-------. 1 root root 1675 10月 15 14:25 etcd-peer-key.pem
      -rw-r--r--. 1 root root 1428 10月 15 14:25 etcd-peer.pem
```

  

* 获取etcd 安装文件

  [ETCD github](  https://github.com/etcd-io/etcd/tags)

* 添加etcd 用户,创建相关目录

   ```
  useradd -s /sbin/nologin -M etcd
  mkdir -p /opt/etcd/certs /data/etcd /data/logs/etcd-server
  	证书目录：certs
  	数据目录：/data/etcd
  	日志目录：/data/logs/etcd-server
  	
  修改属主属组
  chown -R etcd.etcd /data/etcd /data/logs/etcd-server /opt/etcd
  
  复制证书到/opt/etcd/certs
  
  scp 10.4.7.200:/opt/certs/etcd-peer-key.pem etcd-peer.pem  ca.pem 
  
  修改权限
  	chmod 600 ./*.pem
  	
   ```

  

* 创建etcd 启动脚本

  ```
  vim /opt/etcd/etcd-server-startup.sh
  
  ./etcd --name etcd-server-7-12 \
         --data-dir /data/etcd/etcd-server \
         --listen-peer-urls https://10.4.7.12:2380 \
         --listen-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
         --quota-backend-bytes 8000000000 \
         --initial-advertise-peer-urls https://10.4.7.12:2380 \
         --advertise-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
         --initial-cluster  etcd-server-7-12=https://10.4.7.12:2380,etcd-server-7-21=https://10.4.7.21:2380,etcd-server-7-22=https://10.4.7.22:2380 \
         --ca-file ./certs/ca.pem \
         --cert-file ./certs/etcd-peer.pem \
         --key-file ./certs/etcd-peer-key.pem \
         --client-cert-auth  \
         --trusted-ca-file ./certs/ca.pem \
         --peer-ca-file ./certs/ca.pem \
         --peer-cert-file ./certs/etcd-peer.pem \
         --peer-key-file ./certs/etcd-peer-key.pem \
         --peer-client-cert-auth \
         --peer-trusted-ca-file ./certs/ca.pem \
         --log-output stdout
  ```

  

* 编辑supervisor 的etcd 配置文件( hdss7-12)

  ```
  vim /etc/supervisord.d/etcd-server.ini
  
  [program:etcd-server-7-12]
  command=/opt/etcd/etcd-server-startup.sh                        ; the program (relative uses PATH, can take args)
  numprocs=1                                                      ; number of processes copies to start (def 1)
  directory=/opt/etcd                                             ; directory to cwd to before exec (def no cwd)
  autostart=true                                                  ; start at supervisord start (default: true)
  autorestart=true                                                ; retstart at unexpected quit (default: true)
  startsecs=30                                                    ; number of secs prog must stay running (def. 1)
  startretries=3                                                  ; max # of serial start failures (default 3)
  exitcodes=0,2                                                   ; 'expected' exit codes for process (default 0,2)
  stopsignal=QUIT                                                 ; signal used to kill process (default TERM)
  stopwaitsecs=10                                                 ; max num secs to wait b4 SIGKILL (default 10)
  user=etcd                                                       ; setuid to this UNIX account to run the program
  redirect_stderr=true                                            ; redirect proc stderr to stdout (default false)
  stdout_logfile=/data/logs/etcd-server/etcd.stdout.log           ; stdout log path, NONE for none; default AUTO
  stdout_logfile_maxbytes=64MB                                    ; max # logfile bytes b4 rotation (default 50MB)
  stdout_logfile_backups=4                                        ; # of stdout logfile backups (default 10)
  stdout_capture_maxbytes=1MB                                     ; number of bytes in 'capturemode' (default 0)
  stdout_events_enabled=false   
  
  ```


**注意：**
>  **ETCD 的安装需要修改安装文件属主属组（etcd****）。
  