# Service_disc
* 部署K8S内网资源配置清单HTTP 服务

  >HDSS7-200 添加一个NGINX虚拟主机，用于提供K8S统一的资源配置清单入口
  >
  >

  *  配置nginx

    ```
    vim /etc/nginx/conf.d/k8s-yaml.od.com.conf
    
    server{
    	listen	80;
    	server_name	k8s-ymal.od.com;
    	
    	location / {
    		autoindex on;
    		default_type  text/plain;
    		root /data/k8s-yaml;
    	
    	}
    
    }
    
    mkdir /data/k8s-yaml
    ```

  * 配置内网DNS 解析

    ` HDss7-11`

    ```
    vim /var/named/od.com.zone
    
    $ORIGIN od.com.
    $TTL 600        ; 10 minutes
    @               IN SOA  dns.od.com. dnsadmin.od.com. (
                                    2019111003 ; serial   #序号前滚+1
                                    10800      ; refresh (3 hours)
                                    900        ; retry (15 minutes)
                                    604800     ; expire (1 week)
                                    86400      ; minimum (1 day)
                                    )
                                    NS   dns.od.com.
    $TTL 60 ; 1 minute
    dns                A    10.4.7.11
    harbor             A    10.4.7.200
    k8s-ymal		   A	10.4.7.200   #新增
    ```

  * 重启服务 & 测试

    ```
    systemctl restart named
    dig -t A k8s-yaml.od.com @10.4.7.11 +short
    ```

    

* 获取CroeDNS docker镜像

  * HDSS7-200

    ```
    docker pull coredns/coredns:1.6.1
    
    docker tag iamge_s harbor.od.com/public/coredns:v1.6.1
    
    docker push harbor.od.com/public/coredns:v1.6.1
    
    ```

* 配置资源清单

  `mkdir /data/k8s-yaml/coredns &&  cd /data/k8s-yaml/coredns  `

 [模板](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/coredns/coredns.yaml.base)
    
  * RBAC.yaml

    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: coredns
      namespace: kube-system
      labels:
          kubernetes.io/cluster-service: "true"
          addonmanager.kubernetes.io/mode: Reconcile
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      labels:
        kubernetes.io/bootstrapping: rbac-defaults
        addonmanager.kubernetes.io/mode: Reconcile
      name: system:coredns
    rules:
    - apiGroups:
      - ""
      resources:
      - endpoints
      - services
      - pods
      - namespaces
      verbs:
      - list
      - watch
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      annotations:
        rbac.authorization.kubernetes.io/autoupdate: "true"
      labels:
        kubernetes.io/bootstrapping: rbac-defaults
        addonmanager.kubernetes.io/mode: EnsureExists
      name: system:coredns
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:coredns
    subjects:
    - kind: ServiceAccount
      name: coredns
      namespace: kube-system
    ```
* ConfigMAP.yaml

    ```
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: coredns
      namespace: kube-system
      labels:
          addonmanager.kubernetes.io/mode: EnsureExists
    data:
      Corefile: |
        .:53 {
            errors
            health 
            ready
            kubernetes cluster.local 192.168.0.0/16
            forward . 10.4.7.11
            cache 30
            loop
            reload
            loadbalance
        }
    ```

* DeployMent.yaml

    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: coredns
      namespace: kube-system
      labels:
        k8s-app: kube-dns
        kubernetes.io/name: "CoreDNS"
    spec:
      replicas: 1
      selector:
        matchLabels:
          k8s-app: kube-dns
      template:
        metadata:
          labels:
            k8s-app: kube-dns
        spec:
          priorityClassName: system-cluster-critical
          serviceAccountName: coredns
          containers:
          - name: coredns
            image: harbor.od.com/public/coredns:v1.6.1
            args: [ "-conf", "/etc/coredns/Corefile" ]
            volumeMounts:
            - name: config-volume
              mountPath: /etc/coredns
              readOnly: true
            ports:
            - containerPort: 53
              name: dns
              protocol: UDP
            - containerPort: 53
              name: dns-tcp
              protocol: TCP
            - containerPort: 9153
              name: metrics
              protocol: TCP
            livenessProbe:
              httpGet:
                path: /health
                port: 8080
                scheme: HTTP
              initialDelaySeconds: 60
              timeoutSeconds: 5
              successThreshold: 1
              failureThreshold: 5
          dnsPolicy: Default
          volumes:
            - name: config-volume
              configMap:
                name: coredns
                items:
                - key: Corefile
                  path: Corefile
    ```

* svc.yaml

    ```
    apiVersion: v1
    kind: Service
    metadata:
      name: kube-dns
      namespace: kube-system
      labels:
        k8s-app: kube-dns
        kubernetes.io/cluster-service: "true"
        kubernetes.io/name: "CoreDNS"
    spec:
      selector:
        k8s-app: kube-dns
      clusterIP: 192.168.0.2
      ports:
      - name: dns
        port: 53
        protocol: UDP
      - name: dns-tcp
        port: 53
        protocol: TCP
      - name: metrics
        port: 9153
        protocol: TCP
    
    ```

    * 创建相关资源,依次次执行一下命令
    
    ```
          kubectl apply -f http://k8s-yaml.od.com/coredns/rbac.yaml
          kubectl apply -f http://k8s-yaml.od.com/coredns/map.yaml
          kubectl apply -f http://k8s-yaml.od.com/coredns/dp.yaml
          kubectl apply -f http://k8s-yaml.od.com/coredns/svc.yaml
    ```
    * 验证配置结果
    
    ```
        kubectl get -all -n kube-system
        
        [root@hdss7-21 ~]# kubectl get all -n kube-system
        NAME                          READY   STATUS    RESTARTS   AGE
        pod/coredns-96fd597db-4rfbt   1/1     Running   0          8s


        NAME               TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
        service/kube-dns   ClusterIP   192.168.0.2   <none>        53/UDP,53/TCP,9153/TCP   15m


        NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/coredns   1/1     1            1           8s

        NAME                                DESIRED   CURRENT   READY   AGE
        replicaset.apps/coredns-96fd597db   1         1         1       8s
    ```

**注意**
    资源存放的命名空间是 kube-system ,所以在查看资源是否正常的时候应该带上-n 参数