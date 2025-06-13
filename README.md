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

Были ошибки для примеров с интернета, просто взял шаблон values.yaml, который шел с Chart-м и правил его руками, секции что нужны
```
student:~/helm$ cat values.yaml|egrep -iv "##"
# Copyright Broadcom, Inc. All Rights Reserved.
# SPDX-License-Identifier: APACHE-2.0

global:
  imageRegistry: ""
  imagePullSecrets: []
  defaultStorageClass: ""
  storageClass: ""
  security:
    allowInsecureImages: false
  postgresql:
    fullnameOverride: ""
    
    auth:
      postgresPassword: "password"
      username: "esartisonus"
      password: "password"
      database: "esartisondb"
      existingSecret: ""
      secretKeys:
        adminPasswordKey: ""
        userPasswordKey: ""
        replicationPasswordKey: ""
    service:
      ports:
        postgresql: ""
  compatibility:
    openshift:
      adaptSecurityContext: auto

kubeVersion: ""
nameOverride: ""
fullnameOverride: ""
namespaceOverride: ""
clusterDomain: cluster.local
extraDeploy: []
commonLabels: {}
commonAnnotations: {}
secretAnnotations: {}
diagnosticMode:
  enabled: false
  command:
    - sleep
  args:
    - infinity

image:
  registry: docker.io
  repository: bitnami/postgresql
  tag: 17.5.0-debian-12-r11
  digest: ""
  pullPolicy: IfNotPresent
  pullSecrets: []
  debug: false
auth:
  enablePostgresUser: true
  postgresPassword: ""
  username: ""
  password: ""
  database: ""
  replicationUsername: repl_user
  replicationPassword: "replpwd"
  existingSecret: ""
  secretKeys:
    adminPasswordKey: postgres-password
    userPasswordKey: password
    replicationPasswordKey: replication-password
  usePasswordFiles: true
architecture: replication 
replication:
  synchronousCommit: "off"
  numSynchronousReplicas: 2 
  applicationName: my_application
containerPorts:
  postgresql: 5432
audit:
  logHostname: false
  logConnections: false
  logDisconnections: false
  pgAuditLog: ""
  pgAuditLogCatalog: "off"
  clientMinMessages: error
  logLinePrefix: ""
  logTimezone: ""

  args: []
  livenessProbe:
    enabled: true
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
    successThreshold: 1
  readinessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
    successThreshold: 1
  startupProbe:
    enabled: false
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 1
    failureThreshold: 15
    successThreshold: 1
  customLivenessProbe: {}
  customReadinessProbe: {}
  customStartupProbe: {}
  lifecycleHooks: {}
  resourcesPreset: "nano"
  resources: {}
  podSecurityContext:
    enabled: true
    fsGroupChangePolicy: Always
    sysctls: []
    supplementalGroups: []
    fsGroup: 1001
  containerSecurityContext:
    enabled: true
    seLinuxOptions: {}
    runAsUser: 1001
    runAsGroup: 1001
    runAsNonRoot: true
    privileged: false
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop: ["ALL"]
    seccompProfile:
      type: "RuntimeDefault"
  automountServiceAccountToken: false
  hostAliases: []
  hostNetwork: false
  hostIPC: false
  labels: {}
  annotations: {}
  podLabels: {}
  podAnnotations: {}
  podAffinityPreset: ""
  podAntiAffinityPreset: soft
  nodeAffinityPreset:
    type: ""
    key: ""
    values: []
  affinity: {}
  nodeSelector: {}
  tolerations: []
  topologySpreadConstraints: []
  priorityClassName: ""
  schedulerName: ""
  terminationGracePeriodSeconds: ""
  updateStrategy:
    type: RollingUpdate
    rollingUpdate: {}
  extraVolumeMounts: []
  extraVolumes: []
  sidecars: []
  initContainers: []
  pdb:
    create: true
    minAvailable: ""
    maxUnavailable: ""
  extraPodSpec: {}
  networkPolicy:
    enabled: true
    allowExternal: true
    allowExternalEgress: true
    extraIngress: []
    extraEgress: []
    ingressNSMatchLabels: {}
    ingressNSPodMatchLabels: {}
  service:
    type: ClusterIP
    ports:
      postgresql: 5432
    nodePorts:
      postgresql: ""
    clusterIP: ""
    labels: {}
    annotations: {}
    loadBalancerClass: ""
    loadBalancerIP: ""
    externalTrafficPolicy: Cluster
    loadBalancerSourceRanges: []
    extraPorts: []
    sessionAffinity: None
    sessionAffinityConfig: {}
    headless:
      annotations: {}
  persistence:
    enabled: true
    volumeName: "data"
    existingClaim: ""
    mountPath: /bitnami/postgresql
    subPath: ""
    storageClass: ""
    accessModes:
      - ReadWriteOnce
    size: 8Gi
    annotations: {}
    labels: {}
    selector: {}
    dataSource: {}
  persistentVolumeClaimRetentionPolicy:
    enabled: false
    whenScaled: Retain
    whenDeleted: Retain
readReplicas:
  name: read
  replicaCount: 2 
  extendedConfiguration: ""
  extraEnvVars: []
  extraEnvVarsCM: ""
  extraEnvVarsSecret: ""
  command: []
  args: []
  livenessProbe:
    enabled: true
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
    successThreshold: 1
  readinessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
    successThreshold: 1
  startupProbe:
    enabled: false
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 1
    failureThreshold: 15
    successThreshold: 1
  customLivenessProbe: {}
  customReadinessProbe: {}
  customStartupProbe: {}
  lifecycleHooks: {}
  resourcesPreset: "nano"
  resources: {}
  podSecurityContext:
    enabled: true
    fsGroupChangePolicy: Always
    sysctls: []
    supplementalGroups: []
    fsGroup: 1001
  containerSecurityContext:
    enabled: true
    seLinuxOptions: {}
    runAsUser: 1001
    runAsGroup: 1001
    runAsNonRoot: true
    privileged: false
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false

  podAffinityPreset: ""
  podAntiAffinityPreset: soft
  nodeAffinityPreset:
    type: ""
    key: ""
    values: []
  affinity: {}
  nodeSelector: {}
  tolerations: []
  topologySpreadConstraints: []
  priorityClassName: ""
  schedulerName: ""
  terminationGracePeriodSeconds: ""
  updateStrategy:
    type: RollingUpdate
    rollingUpdate: {}
  extraVolumeMounts: []
  extraVolumes: []
  sidecars: []
  initContainers: []
  pdb:
    create: true
    minAvailable: ""
    maxUnavailable: ""
  extraPodSpec: {}
  networkPolicy:
    enabled: true
    allowExternal: true
    allowExternalEgress: true
    extraIngress: []
    extraEgress: []
    ingressNSMatchLabels: {}
    ingressNSPodMatchLabels: {}
  service:
    type: ClusterIP
    ports:
      postgresql: 5432
    nodePorts:
      postgresql: ""
    clusterIP: ""
    labels: {}
    annotations: {}
    loadBalancerClass: ""
    loadBalancerIP: ""
    externalTrafficPolicy: Cluster
    loadBalancerSourceRanges: []
    extraPorts: []
    sessionAffinity: None
    sessionAffinityConfig: {}
    headless:
      annotations: {}
  persistence:
    enabled: true
    existingClaim: ""
    mountPath: /bitnami/postgresql
    subPath: ""
    storageClass: ""
    accessModes:
      - ReadWriteOnce
    size: 8Gi
    annotations: {}
    labels: {}
    selector: {}
    dataSource: {}
  persistentVolumeClaimRetentionPolicy:
    enabled: false
    whenScaled: Retain
    whenDeleted: Retain
backup:
  enabled: false
  cronjob:
    schedule: "@daily"
    timeZone: ""
    concurrencyPolicy: Allow
    failedJobsHistoryLimit: 1
    successfulJobsHistoryLimit: 3
    startingDeadlineSeconds: ""
    ttlSecondsAfterFinished: ""
    restartPolicy: OnFailure
    podSecurityContext:
      enabled: true
      fsGroupChangePolicy: Always
      sysctls: []
      supplementalGroups: []
      fsGroup: 1001
    containerSecurityContext:
      enabled: true
      seLinuxOptions: {}
      runAsUser: 1001
      runAsGroup: 1001
      runAsNonRoot: true
      privileged: false
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      seccompProfile:
        type: "RuntimeDefault"
    command:
      - /bin/bash
      - -c
      - PGPASSWORD="${PGPASSWORD:-$(< "$PGPASSWORD_FILE")}" pg_dumpall --clean --if-exists --load-via-partition-root --quote-all-identifiers --no-password --file="${PGDUMP_DIR}/pg_dumpall-$(date '+%Y-%m-%d-%H-%M').pgdump"
    labels: {}
    annotations: {}
    nodeSelector: {}
    tolerations: []
    resourcesPreset: "nano"
    resources: {}
    networkPolicy:
      enabled: true
    storage:
      enabled: true
      existingClaim: ""
      resourcePolicy: ""
      storageClass: ""
      accessModes:
        - ReadWriteOnce
      size: 8Gi
      annotations: {}
      mountPath: /backup/pgdump
      subPath: ""
      volumeClaimTemplates:
        selector: {}
    extraVolumeMounts: []
    extraVolumes: []

passwordUpdateJob:
  enabled: false
  backoffLimit: 10
  command: []
  args: []
  extraCommands: ""
  previousPasswords:
    postgresPassword: ""
    password: ""
    replicationPassword: ""
    existingSecret: ""
  containerSecurityContext:
    enabled: true
    seLinuxOptions: {}
    runAsUser: 1001
    runAsGroup: 1001
    runAsNonRoot: true
    privileged: false
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop: ["ALL"]
    seccompProfile:
      type: "RuntimeDefault"
  podSecurityContext:
    enabled: true
    fsGroupChangePolicy: Always
    sysctls: []
    supplementalGroups: []
    fsGroup: 1001
volumePermissions:
  enabled: false
  image:
    registry: docker.io
    repository: bitnami/os-shell
    tag: 12-debian-12-r46
    digest: ""
    pullPolicy: IfNotPresent
    pullSecrets: []
  resourcesPreset: "nano"
  resources: {}
  containerSecurityContext:
    seLinuxOptions: {}
    runAsUser: 0
    runAsGroup: 0
    runAsNonRoot: false
    seccompProfile:
      type: RuntimeDefault

serviceBindings:
  enabled: false
serviceAccount:
  create: true
  name: ""
  automountServiceAccountToken: false
  annotations: {}
rbac:
  create: false
  rules: []
psp:
  create: false
metrics:
  enabled: false
  image:
    registry: docker.io
    repository: bitnami/postgres-exporter
    tag: 0.17.1-debian-12-r10
    digest: ""
    pullPolicy: IfNotPresent
    pullSecrets: []
  collectors: {}
  customMetrics: {}
  extraEnvVars: []
  containerSecurityContext:
    enabled: true
    seLinuxOptions: {}
    runAsUser: 1001
    runAsGroup: 1001
    runAsNonRoot: true
    privileged: false
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop: ["ALL"]
    seccompProfile:
      type: "RuntimeDefault"
  livenessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
    successThreshold: 1
  readinessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
    successThreshold: 1
  startupProbe:
    enabled: false
    initialDelaySeconds: 10
    periodSeconds: 10
    timeoutSeconds: 1
    failureThreshold: 15
    successThreshold: 1
  customLivenessProbe: {}
  customReadinessProbe: {}
  customStartupProbe: {}
  containerPorts:
    metrics: 9187
  resourcesPreset: "nano"
  resources: {}
  service:
    ports:
      metrics: 9187
    clusterIP: ""
    sessionAffinity: None
```

установить Helm Chart с Postgres 14й версии
```
student:~/helm$ helm install pges bitnami/postgresql --version 14.3.3 -f values.yaml
NAME: pges
LAST DEPLOYED: Fri Jun 13 20:48:42 2025
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

    pges-postgresql-primary.default.svc.cluster.local - Read/Write connection

    pges-postgresql-read.default.svc.cluster.local - Read only connection

To get the password for "postgres" run:

    export POSTGRES_ADMIN_PASSWORD=$(kubectl get secret --namespace default pges-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To get the password for "esartisonus" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default pges-postgresql -o jsonpath="{.data.password}" | base64 -d)

To connect to your database run the following command:

    kubectl run pges-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:17.5.0-debian-12-r11 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host pges-postgresql-primary -U esartisonus -d esartisondb -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/pges-postgresql-primary 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U esartisonus -d esartisondb -p 5432

WARNING: The configured password will be ignored on new installation in case when previous PostgreSQL release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - primary.resources
  - readReplicas.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```

Поды находятся в рабочем состоянии, 1 праймари и 2 реплики
![image](https://github.com/user-attachments/assets/99708e8b-a639-49eb-bc6d-a61b1dadb25d)




**Укажите параметры подключения в values.yaml.**
в values.yaml указал необходимые пароли изначально
```
    auth:
      postgresPassword: "password"
      username: "esartisonus"
      password: "password"
      database: "esartisondb"
      existingSecret: ""
replicationUsername: repl_user
  replicationPassword: "replpwd"
```



**Обеспечьте масштабируемость: задайте replicaCount: 3 или используйте StatefulSet, если уверены.**

сделать правку в values.yaml и увеличить кол-во replicaCount
```
udent:~/helm$ diff values.yaml values3.yaml
854c854
<   replicaCount: 2 
---
>   replicaCount: 3 
```

применить изменения
```
student:~/helm$ helm upgrade pges bitnami/postgresql --version 14.3.3 --values values3.yaml 
Release "pges" has been upgraded. Happy Helming!
NAME: pges
LAST DEPLOYED: Fri Jun 13 20:52:54 2025
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 14.3.3
APP VERSION: 16.2.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    pges-postgresql-primary.default.svc.cluster.local - Read/Write connection

    pges-postgresql-read.default.svc.cluster.local - Read only connection

To get the password for "postgres" run:

    export POSTGRES_ADMIN_PASSWORD=$(kubectl get secret --namespace default pges-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To get the password for "esartisonus" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default pges-postgresql -o jsonpath="{.data.password}" | base64 -d)

To connect to your database run the following command:

    kubectl run pges-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:17.5.0-debian-12-r11 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host pges-postgresql-primary -U esartisonus -d esartisondb -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/pges-postgresql-primary 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U esartisonus -d esartisondb -p 5432

WARNING: The configured password will be ignored on new installation in case when previous PostgreSQL release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - primary.resources
  - readReplicas.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```

Еще одна релика добавилась как мы и ожидали
![image](https://github.com/user-attachments/assets/860a2da6-1b4f-44bb-abf6-57556f4912ff)


## 🔥 Кризисный момент ## 

**Увеличьте количество подов до 3**
сделано
**Убедитесь, что все поды находятся в статусе Running**
все поды в рабочем состоянии

## 📎 Что сдавать ## 
**📷 Скриншот результата SELECT-запроса из PostgreSQL**
На праймари
![image](https://github.com/user-attachments/assets/7f59e32a-e0c2-42d6-a129-479119874d0b)

На реплике
![image](https://github.com/user-attachments/assets/48c00234-888b-4ff1-8bd4-20862b432305)
![image](https://github.com/user-attachments/assets/0411f882-603e-4782-bb92-e40921d2d2d9)


**📷 Скриншот вывода kubectl get pods (Показывать пароль не нужно!)**
Изначально когда создали кластер
![image](https://github.com/user-attachments/assets/6a77a170-bacb-4564-a1e0-f9a2cf404ae4)


после увеличения replicaCount
![image](https://github.com/user-attachments/assets/17502b34-6be0-4d21-aba7-62e5cdc20127)
