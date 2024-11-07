# Nginx - Flask - MySQL
This application is based on the following doc.  
https://avinton.com/academy/deploying-a-sample-application-on-kubernetes/

And the above doc uses awsome-compose. Here is forked repository. I have fixed a small problem contained in the original repository.
https://github.com/p-rex/sandbag-awesome-compose



# Steps
## DB

### create resources
```:bash
k apply -f mariadb-secret.yaml
k apply -f mariadb-sts.yaml
```


### validate
```
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql# k exec -it mariadb-sts-0 -- mariadb -uroot -pdb-78n9n -e "SHOW DATABASES;"
+--------------------+
| Database           |
+--------------------+
| example            |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql#
```


## Backend
*You can **skip** build steps since there is a builded image in docker hub.*  

### build
Build docker image in no SSL-Proxy environment. 
```bash:build
git clone https://github.com/p-rex/sandbag-awesome-compose.git
cd sandbag-awesome-compose/nginx-flask-mysql/backend
docker build -t prex55/tutrial-nginx-flask-mysql-backend:1.0 . 
docker push prex55/tutrial-nginx-flask-mysql-backend:1.0
```

### create resources
```
k apply -f backend-deployment.yaml
k apply -f backend-service.yaml
```


## nginx
```
 k apply -f nginx-configmap.yaml
 k apply -f nginx-deployment.yaml
 k apply -f nginx-service.yaml
```


# validate
## created resources
```
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql# k get po,deployment,sts,svc,secrets,cm
NAME                                      READY   STATUS    RESTARTS   AGE
pod/mariadb-sts-0                         1/1     Running   0          88m
pod/backend-deployment-866895c5b6-hhmrb   1/1     Running   0          10m
pod/nginx-deployment-b8dbccdd4-qtz42      1/1     Running   0          103s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-deployment   1/1     1            1           10m
deployment.apps/nginx-deployment     1/1     1            1           103s

NAME                           READY   AGE
statefulset.apps/mariadb-sts   1/1     88m

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP        524d
service/db           ClusterIP   None             <none>        3306/TCP       88m
service/backend      ClusterIP   10.152.183.207   <none>        8000/TCP       8m8s
service/proxy        NodePort    10.152.183.220   <none>        80:30080/TCP   57s

NAME                                                   TYPE                             DATA   AGE
secret/mariadb-secret                                  Opaque                           1      88m

NAME                         DATA   AGE
configmap/kube-root-ca.crt   1      524d
configmap/nginx-config       1      2m34s
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql#
```

## access

```
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql# k get svc proxy
NAME    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
proxy   NodePort   10.152.183.220   <none>        80:30080/TCP   2m15s
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql#
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql# k get node -o wide
NAME             STATUS   ROLES    AGE    VERSION    INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
ub22-mk8s-helm   Ready    <none>   524d   v1.26.15   192.168.1.112   <none>        Ubuntu 22.04.1 LTS   5.15.0-43-generic   containerd://1.6.28
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql#
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql# curl 192.168.1.112:30080
<div>   Hello  Blog post #1</div><div>   Hello  Blog post #2</div><div>   Hello  Blog post #3</div><div>   Hello  Blog post #4</div>root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql#
```


## logs
```
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql# k logs nginx-deployment-b8dbccdd4-qtz42
192.168.1.112 - - [06/Nov/2024:02:06:54 +0000] "GET / HTTP/1.1" 200 132 "-" "curl/7.81.0" "-"
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql#
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql# k logs backend-deployment-866895c5b6-hhmrb
 * Serving Flask app 'hello.py' (lazy loading)
 * Environment: development
 * Debug mode: on
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://10.1.130.134:8000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 128-223-970
10.1.130.141 - - [06/Nov/2024 02:06:54] "GET / HTTP/1.0" 200 -
root@ub22-mk8s-helm:~/k8s_nginx-flask-mysql#
```
