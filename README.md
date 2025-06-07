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
kubectl get deployments



**Укажите имя пользователя и пароль через переменные окружения в Deployment (например, POSTGRES_USER и POSTGRES_PASSWORD).**


**Убедитесь, что база данных поднимается и отвечает на подключения (kubectl port-forward + psql).**
kubectl port-forward --namespace default svc/psql-test-postgresql 5432:5432

## ⭐ Задание повышенной сложности ##
## Шаг 2: Развернуть PostgreSQL через Helm ##
https://phoenixnap.com/kb/postgresql-kubernetes
**Установите Helm.**

**Найдите и установите подходящий Helm-чарт PostgreSQL 14 (например, Bitnami PostgreSQL).**

**Укажите параметры подключения в values.yaml.**

**Обеспечьте масштабируемость: задайте replicaCount: 3 или используйте StatefulSet, если уверены.**


## 🔥 Кризисный момент ## 

**Увеличьте количество подов до 3**

**Убедитесь, что все поды находятся в статусе Running**


## 📎 Что сдавать ## 

**📷 Скриншот результата SELECT-запроса из PostgreSQL**

**📷 Скриншот вывода kubectl get pods (Показывать пароль не нужно!)**
