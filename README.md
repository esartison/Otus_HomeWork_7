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
добавить репозиторий, для оригинального поймал ошибку - использовал российское зеркало
```
student:~$ helm repo add bitnami https://charts.bitnami.com/bitnami
Error: looks like "https://charts.bitnami.com/bitnami" is not a valid chart repository or cannot be reached: failed to fetch https://charts.bitnami.com/bitnami/index.yaml : 403 Forbidden

student:~$ helm repo add bitnami  https://mirror.yandex.ru/helm/charts.bitnami.com
"bitnami" has been added to your repositories
student:~$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "postgres-operator-charts" chart repository
...Successfully got an update from the "kubernetes-dashboard" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```

создать PV
```
student:~/helm$ cat postgres-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"

student:~/helm$ kubectl apply -f postgres-pv.yaml
persistentvolume/postgresql-pv created

student:~/helm$ kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
postgresql-pv   10Gi       RWO            Retain           Available           manual         <unset>                          52s
```


создать PVC
```
student:~/helm$ cat postgres-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
student:~/helm$ kubectl apply -f postgres-pvc.yaml
persistentvolumeclaim/postgresql-pv-claim created
student:~/helm$ kubectl get pvc
NAME                  STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
postgresql-pv-claim   Bound    postgresql-pv   10Gi       RWO            manual         <unset>                 12s
```

установить Helm Chart
```
student:~/helm$ helm install psql-test bitnami/postgresql  --version 14.3.3 --set persistence.existingClaim=postgresql-pv-claim --set volumePermissions.enabled=true
NAME: psql-test
LAST DEPLOYED: Fri Jun 13 16:05:00 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 14.3.3
APP VERSION: 16.2.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    psql-test-postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default psql-test-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run psql-test-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.2.0-debian-12-r8 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host psql-test-postgresql -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/psql-test-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

WARNING: The configured password will be ignored on new installation in case when previous PostgreSQL release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - primary.resources
  - readReplicas.resources
  - volumePermissions.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```



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
