# Ingree_Plugin
# Ingress 安装（traefik）

>安装步骤：
>
>1、获取相关镜像；
>
>2、准备资源清单
>
>3、依次应用资源清单
>
>



* 获取 traefik 镜像文件

  * 拉取镜像

    ```
    docker pull traefik:v1.7.2-alpine
    
    ```

  * 修改tag

    ```
    docker image ls  | grep traefik
    docker tag add5fac61ae5 harbor.od.com/public/traefik:v1.7.2
    ```

  * 推送到自建docker镜像仓库

    ```
    docker push harbor.od.com/public/traefik:v1.7.2
    ```



* 准备资源配置清单

  * rbac.yaml

    ```
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: traefik-ingress-controller
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRole
    metadata:
      name: traefik-ingress-controller
    rules:
      - apiGroups:
          - ""
        resources:
          - services
          - endpoints
          - secrets
        verbs:
          - get
          - list
          - watch
      - apiGroups:
          - extensions
        resources:
          - ingresses
        verbs:
          - get
          - list
          - watch
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
      name: traefik-ingress-controller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: traefik-ingress-controller
    subjects:
    - kind: ServiceAccount
      name: traefik-ingress-controller
      namespace: kube-system
    ```

    

  * ds.yaml

```
    apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress
        name: traefik-ingress
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: harbor.zyq.com/public/traefik:v1.7.2
        name: traefik-ingress
        ports:
        - name: controller
          containerPort: 80
          hostPort: 81
        - name: admin-web
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --insecureskipverify=true
        - --kubernetes.endpoint=https://10.4.7.10:7443
        - --accesslog
        - --accesslog.filepath=/var/log/traefik_access.log
        - --traefiklog
        - --traefiklog.filepath=/var/log/traefik.log
        - --metrics.prometheus
```

* svc.yaml

```
    kind: Service
    apiVersion: v1
    metadata:
      name: traefik-ingress-service
      namespace: kube-system
    spec:
      selector:
        k8s-app: traefik-ingress
      ports:
        - protocol: TCP
          port: 80
          name: ingress-controller
        - protocol: TCP
          port: 8080
          name: admin-web
```

* ingrsee.yaml

```
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: traefik-web-ui
      namespace: kube-system
      annotations:
        kubernetes.io/ingress.class: traefik
    spec:
      rules:
      - host: traefik.od.minikube
        http:
          paths:
          - path: /
            backend:
              serviceName: traefik-ingress-service
              servicePort: 8080
```

  

* 创建资源
```
  kubectl create -f http://k8s-yaml.zq.com/traefik/rbac.yaml
  kubectl create -f http://k8s-yaml.zq.com/traefik/ds.yaml
  kubectl create -f http://k8s-yaml.zq.com/traefik/svc.yaml
  kubectl create -f http://k8s-yaml.zq.com/traefik/ingress.yaml
```

  

* 创建NGINX配置文件

  > vim /etc/nginx/conf.d/zq.com.conf
```
  upstream default_backend_traefik {
      server 10.4.7.21:81    max_fails=3 fail_timeout=10s;
      server 10.4.7.22:81    max_fails=3 fail_timeout=10s;
  }
  server {
      server_name *.od.com;
    
      location / {
          proxy_pass http://default_backend_traefik;
          proxy_set_header Host       $http_host;
          proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
      }
  }
```

  > 重启NGINX
  >
  > systemctl restart nginx

* DNS 解析

  > vim /var/named/od.com.zon

  ```
  traefik            A    10.4.7.10
  ```

  