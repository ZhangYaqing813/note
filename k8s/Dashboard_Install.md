# Dashboard_Install

[官方站点](https://github.com/kubernetes/dashboard/tree/v1.8.3)

## 安装三步走

* **获取镜像文件**

  > 1、获取镜像

  ```
  docker pull k8scn/kubernetes-dashboard-amd64:v1.8.3
  ```

  > 2、打tag,推送到镜像仓库

  ```
  docker tag fcac9aa03fd6 harbor.od.com/public/dashboard:v1.8.3
  ocker push harbor.od.com/public/dashboard:v1.8.3
  ```

* **资源配置清单**

  > 1、RBAC.yaml
  >
  > mkdir /data/k8s-yaml/dashboard && cd /data/k8s-yaml/dashboard
  ```
  apiVsersion: v1
  kind: ServiceAccount
  matedata:
    labels:
      k8s-app: kubernetes-dashboard
      addonmanager.kubernetes.io/mode: Reconcile
    name: ubernetes-dashboard-admin
    namespace: kube-system
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: kubernetes-dashboard-admin
    namespace: kube-system
    labels:
      k8s-app: kubernetes-dashboard
      addonmanager.kubernetes.io/mode: Reconcile
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: Cluster-admin
  subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system
  ```

  > 2、deployment.yaml

  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: kubernetes-dashboard
    namespace: kube-system
    labels:
      k8s-app: kubernetes-dashboard
      kubernetes.io/cluster-service: "ture"
      addonmanager.kubernetes.io/mode: Reconcile
  spec:
    selector:
      matchLabels:
        k8s-app: kubernetes-dashboard
    template:
      metadata:
        labels:
          k8s-app: kubernetes-dashboard
        annotations:
          scheduler.alpha.kubernetes.io/critical-pod: ''
          #seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
      spec:
        priorityClassName: system-cluster-critical
        containers:
        - name: kubernetes-dashboard
          image: harbor.od.com/public/dashboard:v1.8.3
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 50m
              memory: 100Mi
          ports:
          - containerPort: 8443
            protocol: TCP
          args:
            # PLATFORM-SPECIFIC ARGS HERE
            - --auto-generate-certificates
          volumeMounts:
          - name: tmp-volume
            mountPath: /tmp
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
        volumes:
        - name: tmp-volume
          emptyDir: {}
        serviceAccountName: kubernetes-dashboard-admin
        tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
  
  ```
  > 3、svc.yaml

  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: kubernetes-dashboard
    namespace: kube-system
    labels:
      k8s-app: kubernetes-dashboard
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
  spec:
    selector:
      k8s-app: kubernetes-dashboard
    ports:
    - port: 443
      targetPort: 8443
  ```

  > 4、ingress.yaml

  ```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: kubernetes-dashboard
    namespace: kube-system
    annotations:
      kubernetes.io/ingress.class: graefik
  spec:
    rules:
      - host: dashboard.od.com
        http:
          paths:
            - backend:
                serviceName: kubernetes-dashboard
                servicePort: 443
  ```

  > 5、创建资源 hdss7-21

  ```
  kubectl apply -f http://k8s-yaml.od.com/dashboard/rbac.yaml
  kubectl apply -f http://k8s-yaml.od.com/dashboard/ds.yaml
  kubectl apply -f http://k8s-yaml.od.com/dashboard/svc.yaml
  kubectl apply -f http://k8s-yaml.od.com/dashboard/ingress.yaml
  ```

* **域名解析**

  ```
  vim /var/named/od.com.zone
  
  dashboard          A    10.4.7.10
  
  systemctl restart named
  ```

  

* **获取token**

  > kubectl get secret  -n kube-system

  ```
  ~]# kubectl get secret  -n kube-system
  NAME                                     TYPE                                  DATA   AGE
  coredns-token-8djq6                      kubernetes.io/service-account-token   3      16d
  default-token-tfbzh                      kubernetes.io/service-account-token   3      29d
  kubernetes-dashboard-admin-token-vndqx   kubernetes.io/service-account-token   3      6d5h
  kubernetes-dashboard-key-holder          Opaque                                2      6d5h
  traefik-ingress-controller-token-lr6zb   kubernetes.io/service-account-token   3      12d
  ubernetes-dashboard-admin-token-znmdz    kubernetes.io/service-account-token   3      7d
  
  ```

  

  > 查看token详细信息

  ```
  kubectl -n kube-system describe secrets kubernetes-dashboard-admin-token-vndqx
  ```

  

* 申请https 证书

  > 生成域名dashboard.od.com 的私钥文件 

  ```
  (umask 077; openssl genrsa -out dashboard.od.com.key 2048)
  
  Generating RSA private key, 2048 bit long modulus
  ................+++
  ...............+++
  e is 65537 (0x10001)
  
  ```

  > 生成域名dashboard.od.com 的csr 文件

  ```
  openssl req -new -key dashboard.od.com.key -out dashboard.od.com.csr -subj "/CN=dashboard.od.com/C=CN/ST=BJ/L=Beijing/O=OldboyEdu/OU=ops"
  
  certs]# ll
  total 108
  -rw-r--r--. 1 root root 1249 10月 15 17:09 apiserver.csr
  -rw-r--r--. 1 root root  566 10月 15 17:04 apiserver-csr.json
  -rw-------. 1 root root 1675 10月 15 17:09 apiserver-key.pem
  -rw-r--r--. 1 root root 1598 10月 15 17:09 apiserver.pem
  -rw-r--r--. 1 root root  837 10月 15 14:20 ca-config.json
  -rw-r--r--. 1 root root  993 10月 14 17:48 ca.csr
  -rw-r--r--. 1 root root  329 10月 14 17:48 ca-csr.json
  -rw-------. 1 root root 1675 10月 14 17:48 ca-key.pem
  -rw-r--r--. 1 root root 1346 10月 14 17:48 ca.pem
  -rw-r--r--. 1 root root  993 10月 15 17:05 client.csr
  -rw-r--r--. 1 root root  280 10月 15 17:01 client-csr.json
  -rw-------. 1 root root 1675 10月 15 17:05 client-key.pem
  -rw-r--r--. 1 root root 1363 10月 15 17:05 client.pem
  -rw-r--r--. 1 root root 1005 11月 18 17:34 dashboard.od.com.csr
  -rw-------. 1 root root 1675 11月 18 17:31 dashboard.od.com.key
  -rw-r--r--. 1 root root 1062 10月 15 14:25 etcd-peer.csr
  -rw-r--r--. 1 root root  363 10月 15 14:21 etcd-peer-csr.json
  -rw-------. 1 root root 1675 10月 15 14:25 etcd-peer-key.pem
  -rw-r--r--. 1 root root 1428 10月 15 14:25 etcd-peer.pem
  -rw-r--r--. 1 root root 1115 10月 20 16:19 kubelet.csr
  -rw-r--r--. 1 root root  452 10月 20 16:17 kubelet-csr.json
  -rw-------. 1 root root 1675 10月 20 16:19 kubelet-key.pem
  -rw-r--r--. 1 root root 1468 10月 20 16:19 kubelet.pem
  -rw-r--r--. 1 root root 1005 10月 20 18:13 kube-proxy-client.csr
  -rw-------. 1 root root 1679 10月 20 18:13 kube-proxy-client-key.pem
  -rw-r--r--. 1 root root 1375 10月 20 18:13 kube-proxy-client.pem
  -rw-r--r--. 1 root root  267 10月 20 18:11 kube-proxy-csr.json
  
  ```
  
  > 生成域名dashboard.od.com 的crt 文件

  ```
  openssl x509 -req -in dashboard.od.com.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out dashboard.od.com.crt -days 3650
  
  certs]# ll
  total 116
  -rw-r--r--. 1 root root 1249 10月 15 17:09 apiserver.csr
  -rw-r--r--. 1 root root  566 10月 15 17:04 apiserver-csr.json
  -rw-------. 1 root root 1675 10月 15 17:09 apiserver-key.pem
  -rw-r--r--. 1 root root 1598 10月 15 17:09 apiserver.pem
  -rw-r--r--. 1 root root  837 10月 15 14:20 ca-config.json
  -rw-r--r--. 1 root root  993 10月 14 17:48 ca.csr
  -rw-r--r--. 1 root root  329 10月 14 17:48 ca-csr.json
  -rw-------. 1 root root 1675 10月 14 17:48 ca-key.pem
  -rw-r--r--. 1 root root 1346 10月 14 17:48 ca.pem
  -rw-r--r--. 1 root root   17 11月 18 17:37 ca.srl
  -rw-r--r--. 1 root root  993 10月 15 17:05 client.csr
  -rw-r--r--. 1 root root  280 10月 15 17:01 client-csr.json
  -rw-------. 1 root root 1675 10月 15 17:05 client-key.pem
  -rw-r--r--. 1 root root 1363 10月 15 17:05 client.pem
  -rw-r--r--. 1 root root 1196 11月 18 17:37 dashboard.od.com.crt
  -rw-r--r--. 1 root root 1005 11月 18 17:34 dashboard.od.com.csr
  -rw-------. 1 root root 1675 11月 18 17:31 dashboard.od.com.key
  -rw-r--r--. 1 root root 1062 10月 15 14:25 etcd-peer.csr
  -rw-r--r--. 1 root root  363 10月 15 14:21 etcd-peer-csr.json
  -rw-------. 1 root root 1675 10月 15 14:25 etcd-peer-key.pem
  -rw-r--r--. 1 root root 1428 10月 15 14:25 etcd-peer.pem
  -rw-r--r--. 1 root root 1115 10月 20 16:19 kubelet.csr
  -rw-r--r--. 1 root root  452 10月 20 16:17 kubelet-csr.json
  -rw-------. 1 root root 1675 10月 20 16:19 kubelet-key.pem
  -rw-r--r--. 1 root root 1468 10月 20 16:19 kubelet.pem
  -rw-r--r--. 1 root root 1005 10月 20 18:13 kube-proxy-client.csr
  -rw-------. 1 root root 1679 10月 20 18:13 kube-proxy-client-key.pem
  -rw-r--r--. 1 root root 1375 10月 20 18:13 kube-proxy-client.pem
  -rw-r--r--. 1 root root  267 10月 20 18:11 kube-proxy-csr.json
  
  ```

  > 查看生成证书信息 

  ```
  cfssl-certinfo -cert dashboard.od.com.crt
  
  certs]# cfssl-certinfo -cert dashboard.od.com.crt
  {
    "subject": {
      "common_name": "dashboard.od.com",
      "country": "CN",
      "organization": "OldboyEdu",
      "organizational_unit": "ops",
      "locality": "Beijing",
      "province": "BJ",
      "names": [
        "dashboard.od.com",
        "CN",
        "BJ",
        "Beijing",
        "OldboyEdu",
        "ops"
      ]
    },
    "issuer": {
      "common_name": "OldboyEdu",
      "country": "CN",
      "organization": "od",
      "organizational_unit": "ops",
      "locality": "beijing",
      "province": "beijing",
      "names": [
        "CN",
        "beijing",
        "beijing",
        "od",
        "ops",
        "OldboyEdu"
      ]
    },
    "serial_number": "17595613907782020856",
    "not_before": "2020-11-18T09:37:15Z",
    "not_after": "2030-11-16T09:37:15Z",
    "sigalg": "SHA256WithRSA",
    "authority_key_id": "",
    "subject_key_id": "",
    "pem": "-----BEGIN CERTIFICATE-----\nMIIDRTCCAi0CCQD0MC2fokUu+DANBgkqhkiG9w0BAQsFADBgMQswCQYDVQQGEwJD\nTjEQMA4GA1UECBMHYmVpamluZzEQMA4GA1UEBxMHYmVpamluZzELMAkGA1UEChMC\nb2QxDDAKBgNVBAsTA29wczESMBAGA1UEAxMJT2xkYm95RWR1MB4XDTIwMTExODA5\nMzcxNVoXDTMwMTExNjA5MzcxNVowaTEZMBcGA1UEAwwQZGFzaGJvYXJkLm9kLmNv\nbTELMAkGA1UEBhMCQ04xCzAJBgNVBAgMAkJKMRAwDgYDVQQHDAdCZWlqaW5nMRIw\nEAYDVQQKDAlPbGRib3lFZHUxDDAKBgNVBAsMA29wczCCASIwDQYJKoZIhvcNAQEB\nBQADggEPADCCAQoCggEBAKOLaWctP7/EgoKjhqnZ7CnHqQQGJgX7hCtlBQkrgCLv\nveqBZHI4fAazupP0wLNdbqDHA82t0h0dS4UzUq1lcLAgFvbltGk3TvKaokhLhlE9\nWSzUSMwowmPoLnwZKCf7nAW8E9XpsshmHUaVtPFoBTfZYFuIN24tUrdclx4UwyXz\naOyeq0ptjxbTxsY8G+zSUOeOY+Abmua73k8r9yxy0cayTwNvb7VYVvZnN9195I5+\nvJwoyifB/vbGaziBh0EK3OONDn1WzGHT7zMa+atFXPC3pMjiz7ElJkGDdFBSV5qR\n7S3zLteWTAk2+6M/xbBxOFXzvxdA2HFlitTTZDL3Z00CAwEAATANBgkqhkiG9w0B\nAQsFAAOCAQEAE7u/Rl5emE9S//ffdWg3rcutKQ3Nz83VNo2ZQhMrBm448ZArjFY6\n88EJpwh/m1LO1Hx9jRJ/QCEfQLTrr1xKZIm4tShOvLj2x9nhZd5M3tp1n0UzFlcF\nqjF0YolbfZs4WPpgZtuawroQ/0Ax+sibpHj4K/KexZKZV/ciUmk9Ud8nZqS/Rypn\nOlvoUtuNhARyJcjXXwZBD4EgWVmDu2VTh0y+UriFrscdA+Z0j2pEQRc0028HElJj\nHQKfbf8n5+Fqo1mfiPS0OnY6noJu5uax6+HDc+8O9akwqusIhYvS+EKz8O7EzYIT\nYBiKikEDMVGIbdc9Pt3LS8ORuRWrpJ8mPw==\n-----END CERTIFICATE-----\n"
  }
  
  ```

* **Nginx 配置**

  > 获取证书

  ```
  mkdir /etc/nginx/certs
  cd /etc/nginx/certs
  scp 10.4.7.200:/opt/certs/dashboard.od.com.crt ./
  scp 10.4.7.200:/opt/certs/dashboard.od.com.key ./
  ```

  > 编辑NGINX配置文件 (HDSS7-11 && HDSS7-12)
  >
  > vim /etc/nginx/conf.d/dashboard.od.com.conf

  ```
  server {
      listen       80;
      server_name  dashboard.od.com;
  
      rewrite ^(.*)$ https://${server_name}$1 permanent;
  }
  server {
      listen       443 ssl;
      server_name  dashboard.od.com;
  
      ssl_certificate     "certs/dashboard.od.com.crt";
      ssl_certificate_key "certs/dashboard.od.com.key";
      ssl_session_cache shared:SSL:1m;
      ssl_session_timeout  10m;
      ssl_ciphers HIGH:!aNULL:!MD5;
      ssl_prefer_server_ciphers on;
  
      location / {
          proxy_pass http://default_backend_traefik;
          proxy_set_header Host       $http_host;
          proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
      }
  }
  
  ```

  > 重启NGINX

  ```
  nginx -t 
  nginx -s reload
  ```

* 测试

  ```
  https://dashboard.od.com
  ```
