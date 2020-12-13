# Base_Commond
常用命令

* kubectl get namaspace

  获取命名空间

  ```
  [root@hdss7-21 ~]# kubectl get namespace
  NAME              STATUS   AGE
  default           Active   10d
  kube-node-lease   Active   10d
  kube-public       Active   10d
  kube-system       Active   10d
  ```

* kubectl get all [-n deafault]

  查看命名空间中的资源

  ```
  NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
  service/kubernetes   ClusterIP   192.168.0.1   <none>        443/TCP   10d
  
  
  ```

* kubectl create ns app

  创建命名空间

  ```
  [root@hdss7-21 ~]# kubectl create ns app
  namespace/app created
  
  ```

* kubectl create deployment nginx-dp --image=harbor.od.com/public/nginx:v1.7.9 -n kube-public

  创建 deployment 

  ```
  deployment.apps/nginx-dp created
  
  ```

*  kubectl get deploy -n kube-public

  查看创建的资源

  ```
  [root@hdss7-21 ~]# kubectl get deployment -n kube-public
  NAME                        READY   STATUS    RESTARTS   AGE
  nginx-dp-5dfc689474-phx9q   1/1     Running   0          11s
  
  ```

* kubectl get deployment -o wide -n kube-public

  查看所有的deployment 详细信息

  ```
  [root@hdss7-21 ~]# kubectl get deployment -o wide -n kube-public
  NAME       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                              SELECTOR
  nginx-dp   1/1     1            1           91s   nginx        harbor.od.com/public/nginx:v1.7.9   app=nginx-dp
  
  ```

* kubectl get pods -n kube-public

  查看命名空间中所有pod 信息

  ```
  [root@hdss7-21 ~]# kubectl get pods -n kube-public
  NAME                        READY   STATUS    RESTARTS   AGE
  nginx-dp-5dfc689474-phx9q   1/1     Running   0          5m12s
  
  ```

* kubectl exec -ti nginx-dp-5dfc689474-phx9q /bin/bash -n kube-public

  进入pod 内部

  ```
  [root@hdss7-21 ~]# kubectl exec -ti nginx-dp-5dfc689474-phx9q /bin/bash -n kube-public
  root@nginx-dp-5dfc689474-phx9q:/# 
  root@nginx-dp-5dfc689474-phx9q:/# 
  root@nginx-dp-5dfc689474-phx9q:/# 
  root@nginx-dp-5dfc689474-phx9q:/# ifconfig
  bash: ifconfig: command not found
  root@nginx-dp-5dfc689474-phx9q:/# ip add
  bash: ip: command not found
  root@nginx-dp-5dfc689474-phx9q:/# ip addr
  bash: ip: command not found
  root@nginx-dp-5dfc689474-phx9q:/# ls
  bin  boot  dev	docker-entrypoint.d  docker-entrypoint.sh  etc	home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
  root@nginx-dp-5dfc689474-phx9q:/# 
  
  
  ```

*  kubectl delete pod nginx-dp-5dfc689474-phx9q -n kube-public

  删除pod 资源，但是删除后kubelet 会根据用户期望重新创建对应数量的pod资源，这是deployment 特性决定的

  ```
  [root@hdss7-21 ~]# kubectl delete pod nginx-dp-5dfc689474-phx9q -n kube-public
  pod "nginx-dp-5dfc689474-phx9q" deleted
  
  [root@hdss7-21 ~]# 
  [root@hdss7-21 ~]# 
  [root@hdss7-21 ~]# 
  [root@hdss7-21 ~]# kubectl get pods -n kube-public
  NAME                        READY   STATUS    RESTARTS   AGE
  nginx-dp-5dfc689474-76kn8   1/1     Running   0          7s
  [root@hdss7-21 ~]# 
  
  ```

* kubectl delete deployment nginx-dp -n kube-public

  删除deployment 资源，删除后deployment 控制器不会在重新创建相关资源

  ```
  [root@hdss7-21 ~]# kubectl delete deployment nginx-dp -n kube-public
  deployment.extensions "nginx-dp" deleted
  [root@hdss7-21 ~]# kubectl get deploy -n kube-public
  No resources found.
  [root@hdss7-21 ~]# 
  
  ```

* kubectl expose deployment nginx-dp --port=80 -n kube-public

  对集群内部暴露一个服务，相当于创建了一个svc 资源

  ```
  [root@hdss7-21 ~]# kubectl expose deployment nginx-dp --port=80 -n kube-public
  service/nginx-dp exposed
  [root@hdss7-21 ~]# kubectl get svc -n kube-public
  NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
  nginx-dp   ClusterIP   192.168.74.6   <none>        80/TCP    14s
  [root@hdss7-21 ~]# 
  
  ```

* kubectl scale deployment nginx-dp --replicas=2 -n kube-public

  增加对现有资源期望的副本数，对现有资源进行扩容。

  ```
  [root@hdss7-21 ~]# kubectl scale deployment nginx-dp --replicas=2 -n kube-public
  deployment.extensions/nginx-dp scaled
  [root@hdss7-21 ~]# kubectl get deploy -n kube-public 
  NAME       READY   UP-TO-DATE   AVAILABLE   AGE
  nginx-dp   2/2     2            2           2m59s
  [root@hdss7-21 ~]# 
  
  ```

  
    

      
    