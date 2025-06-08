# Домашнее задание Сартисона Евгения №7 #


## 🚧  Подготовка ##

Развернул Minikube на VM с Ubuntu под управлением Virtualbox и выделил ресурсы

![image](https://github.com/user-attachments/assets/d3027109-5ded-48cb-80c4-fbcf7b4408c1)

![image](https://github.com/user-attachments/assets/05e58bc6-67a8-49ec-aadb-c742c0166a2b)


**Установите и запустите Minikube.**

![image](https://github.com/user-attachments/assets/2aae8a08-ace9-4a33-b8a6-b5ee2c3799a2)


**Убедитесь, что у вас установлены kubectl и psql.**

psql стоит 

![image](https://github.com/user-attachments/assets/57e48cc4-0f09-4b07-a67c-3ac731b58957)

kubectl стоит и работает

![image](https://github.com/user-attachments/assets/b1358a70-9a67-4626-a49a-1235dc15509c)


## 🔨 Основная часть ##
## Шаг 1: Развернуть PostgreSQL через манифест ##
https://dev.to/dm8ry/how-to-deploy-postgresql-db-server-and-pgadmin-in-kubernetes-a-how-to-guide-5fm0


**Создайте Deployment и Service для PostgreSQL.**

создать secret и прописать пароль для root схемы(esartison) в postgres кластер
```
student:~/test1$ echo -n 'esartison' | base64
ZXNhcnRpc29u
student:~/test1$ echo -n 'notyourpwd' | base64
bm90eW91cnB3ZA==

tudent:~/test1$ cat postgres-secret.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  labels:
    app: postgres
type: Opaque
data:
    postgres-root-username: ZXNhcnRpc29u
    postgres-root-password: bm90eW91cnB3ZA==
student:~/test1$ kubectl apply -f postgres-secret.yaml
secret/postgres-secret created

student:~/test1$ kubectl get secret
NAME              TYPE     DATA   AGE
postgres-secret   Opaque   2      33s
```

создать configmap и прописать имя базы, которая будет создана в postgres кластере
```
tudent:~/test1$ cat postgres-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-configmap
  labels:
    app: postgres
data:
  postgres-dbname: esartison_db 
student:~/test1$ kubectl apply -f postgres-configmap.yaml
configmap/postgres-configmap created
student:~/test1$ kubectl get configmap
NAME                 DATA   AGE
kube-root-ca.crt     1      4m55s
postgres-configmap   1      16s
```

создать deployment с переменными 
```
student:~/test1$ cat postgres-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
   matchLabels:
    app: postgres
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:latest
          imagePullPolicy: "IfNotPresent"
          env:
           - name: POSTGRES_USER
             valueFrom:
               secretKeyRef:
                 name: postgres-secret
                 key: postgres-root-username
           - name: POSTGRES_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: postgres-secret
                 key: postgres-root-password
           - name: POSTGRES_DB
             valueFrom:
               configMapKeyRef:
                 name: postgres-configmap
                 key: postgres-dbname

          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgredb
      volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: postgres-pv-claim

student:~/test1$ kubectl apply -f postgres-deployment.yaml
deployment.apps/postgres created

student:~/test1$ kubectl get deployments
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
postgres   0/1     1            0           13s
```

создать PV и PVC, чтобы данные не терялись при перезапуске контейнера
```
student:~/test1$ cat postgres_PV_PVC.yaml
---
    kind: PersistentVolume
    apiVersion: v1
    metadata:
      name: postgres-pv-volume
      labels:
        type: local
        app: postgres
    spec:
      storageClassName: manual
      capacity:
        storage: 5Gi
      accessModes:
        - ReadWriteMany
      hostPath:
        path: "/mnt/data"
---
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: postgres-pv-claim
      labels:
        app: postgres
    spec:
      storageClassName: manual
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 5Gi
student:~/test1$ 
student:~/test1$ 
student:~/test1$ kubectl apply -f postgres_PV_PVC.yaml
persistentvolume/postgres-pv-volume created
persistentvolumeclaim/postgres-pv-claim created
student:~/test1$ kubectl get PersistentVolume
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
postgres-pv-volume   5Gi        RWX            Retain           Bound    default/postgres-pv-claim   manual         <unset>                          21s
student:~/test1$ kubectl get PersistentVolumeClaim
NAME                STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
postgres-pv-claim   Bound    postgres-pv-volume   5Gi        RWX            manual         <unset>                 34s

```

Создать сервис для связи контейнеров с внешним миров по сети 
```
student:~/test1$ cat postgres-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: postgres
      labels:
        app: postgres
    spec:
       ports:
        - name: postgres
          port: 5432
          nodePort: 30432
       type: NodePort
       selector:
        app: postgres

student:~/test1$ kubectl apply -f postgres-service.yaml
service/postgres created

student:~/test1$ kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          21m
postgres     NodePort    10.109.208.50   <none>        5432:30432/TCP   56s
```        

**Укажите имя пользователя и пароль через переменные окружения в Deployment (например, POSTGRES_USER и POSTGRES_PASSWORD).**

указал во время создания deployment-а

![image](https://github.com/user-attachments/assets/30ded318-0023-425b-bf6a-4adba74e14cd)

првоерка статуса- все хорошо!
``` 
student:~/test1$ kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          21m
postgres     NodePort    10.109.208.50   <none>        5432:30432/TCP   56s
student:~/test1$  kubectl get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/postgres-7fd78bcdf8-llccq   1/1     Running   0          11m

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          22m
service/postgres     NodePort    10.109.208.50   <none>        5432:30432/TCP   2m45s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/postgres   1/1     1            1           11m

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/postgres-7fd78bcdf8   1         1         1       11m
student:~/test1$ minikube service postgres
|-----------|----------|---------------|---------------------------|
| NAMESPACE |   NAME   |  TARGET PORT  |            URL            |
|-----------|----------|---------------|---------------------------|
| default   | postgres | postgres/5432 | http://192.168.49.2:30432 |
|-----------|----------|---------------|---------------------------|


student:~/test1$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
postgres-7fd78bcdf8-llccq   1/1     Running   0          16m

``` 



**Убедитесь, что база данных поднимается и отвечает на подключения (kubectl port-forward + psql).**

включиить port forward на 5432 в POD-е на 5555 локально
``` 
student:~$ kubectl port-forward --namespace default svc/postgres 5555:5432 
Forwarding from 127.0.0.1:5555 -> 5432
Forwarding from [::1]:5555 -> 5432
Handling connection for 5555
Handling connection for 5555
Handling connection for 5555

```

проверить локальное подключение к postgres кластеру в Kuber под схемой esartison по порту 5555
```
student:~$ read -s POSTGRES_PASSWORD
student:~$ PGPASSWORD="$POSTGRES_PASSWORD" psql -h 127.0.0.1 -p 5555 -U esartison postgres
psql (17.5 (Ubuntu 17.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \l
                                                       List of databases
     Name     |   Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | Locale | ICU Rules >
--------------+-----------+----------+-----------------+------------+------------+--------+----------->
 esartison_db | esartison | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           >
 postgres     | esartison | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           >
 template0    | esartison | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           >
              |           |          |                 |            |            |        |           >
 template1    | esartison | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           >
              |           |          |                 |            |            |        |           >

postgres-# \conninfo
You are connected to database "postgres" as user "esartison" on host "127.0.0.1" at port "5555".

postgres=# SELECT inet_server_port() AS portNumber;
 portnumber 
------------
       5432
```

Все прошло успешно!

## ⭐ Задание повышенной сложности ##
очистить предыдущую конфигурацию в MiniKube
```
student:~$ minikube delete
🔥  Deleting "minikube" in docker ...
🔥  Deleting container "minikube" ...
🔥  Removing /home/student/.minikube/machines/minikube ...
💀  Removed all traces of the "minikube" cluster.
```

запустить minikube с большим кол-вом ресурсов
```
student:~$ minikube start --cpus 4 --memory 4g
😄  minikube v1.36.0 on Ubuntu 22.04 (vbox/amd64)
❗  The minimum required version for podman is "4.9.0". your version is "3.4.4". minikube might not work. use at your own risk. To install latest version please see https://podman.io/getting-started/installation.html
✨  Automatically selected the docker driver. Other choices: podman, none, ssh

🧯  The requested memory allocation of 4096MiB does not leave room for system overhead (total system memory: 4816MiB). You may face stability issues.
💡  Suggestion: Start minikube with less memory allocated: 'minikube start --memory=2200mb'

📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.47 ...
🔥  Creating docker container (CPUs=4, Memory=4096MB) ...
🐳  Preparing Kubernetes v1.33.1 on Docker 28.1.1 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```


## Шаг 2: Развернуть PostgreSQL через Helm ##
https://phoenixnap.com/kb/postgresql-kubernetes
**Установите Helm.**
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

student:~$ helm version
version.BuildInfo{Version:"v3.18.1", GitCommit:"f6f8700a539c18101509434f3b59e6a21402a1b2", GitTreeState:"clean", GoVersion:"go1.24.3"}
```

**Найдите и установите подходящий Helm-чарт PostgreSQL 14 (например, Bitnami PostgreSQL).**

student:~/helm$ helm repo add bitnami  https://mirror.yandex.ru/helm/charts.bitnami.com
"bitnami" has been added to your repositories

```
kubectl create secret generic postgres-credentials -n default \
  --from-literal=POSTGRES_PASSWORD='AdminPassword' \
  --from-literal=APP_DB_PASSWORD='AUserPassword'
```

Создать директории
```
sudo mkdir -p /data/postgres-data /data/postgres-dump

sudo chown -R 1001:1001 /data/postgres-data
sudo chown -R 1001:1001 /data/postgres-dump

sudo chmod -R 750 /data/postgres-data
sudo chmod -R 750 /data/postgres-dump
```


Create Persistent Volumes
```
student:~/helm$ cat postgres-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-data-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  claimRef:
    namespace: database
    name: data-postgres-postgresql-0
  hostPath:
    path: /data/postgres-data
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-dump-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  claimRef:
    namespace: database
    name: postgres-postgresql-pgdumpall
  hostPath:
    path: /data/postgres-dump
    type: DirectoryOrCreate

student:~/helm$ kubectl create -f postgres-pv.yaml
persistentvolume/postgres-data-pv created
persistentvolume/postgres-dump-pv created

```
https://cicube.io/blog/postgres-kubernetes/
https://artifacthub.io/packages/helm/bitnami/postgresql-ha

helm install helmtestrel oci://registry-1.docker.io/bitnamicharts/postgresql-ha --set postgresql.replicaCount=2

**Укажите параметры подключения в values.yaml.**

**Обеспечьте масштабируемость: задайте replicaCount: 3 или используйте StatefulSet, если уверены.**
student:~/helm$ kubectl scale --replicas=3 deployment helmtestrel-postgresql-ha-pgpool -n default
kubectl get deployments
student:~/helm$ kubectl get pod -A


## 🔥 Кризисный момент ## 

**Увеличьте количество подов до 3**

**Убедитесь, что все поды находятся в статусе Running**


## 📎 Что сдавать ## 

**📷 Скриншот результата SELECT-запроса из PostgreSQL**

**📷 Скриншот вывода kubectl get pods (Показывать пароль не нужно!)**
