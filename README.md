# Домашнее задание к занятию «Запуск приложений в K8S»

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.
2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.

------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------

### Решение 1


1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.

 - Ошибка: nginx, и multitool слушают 80/tcp в одном Pod → один из контейнеров падает с address already in use
 - Решение: запуск web-сервера multitool на другом порту

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - name: http-nginx
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http-nginx
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: http-nginx
            initialDelaySeconds: 10
            periodSeconds: 10

        - name: multitool
          image: wbitt/network-multitool:alpine-minimal
          env:
            - name: HTTP_PORT
              value: "8080"
          ports:
            - name: http-mt
              containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: http-mt
            initialDelaySeconds: 3
            periodSeconds: 5

```


![Scr 1](https://github.com/vladrabbit/K8S-2/blob/main/img/1.png)

2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.

![Scr 2](https://github.com/vladrabbit/K8S-2/blob/main/img/2.png)


4. Создать Service, который обеспечит доступ до реплик приложений из п.1.

```yaml

apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
    - name: nginx
      port: 80
      targetPort: http-nginx
      protocol: TCP
    - name: multitool
      port: 8080
      targetPort: http-mt
      protocol: TCP
  type: ClusterIP

```

![Scr 3](https://github.com/vladrabbit/K8S-2/blob/main/img/3.png)

![Scr 4](https://github.com/vladrabbit/K8S-2/blob/main/img/4.png)

5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mt
spec:
  containers:
    - name: mt
      image: wbitt/network-multitool:alpine-minimal
      command: ["sleep","infinity"]
```

![Scr 5](https://github.com/vladrabbit/K8S-2/blob/main/img/5.png)

 - Проверка логов

![Scr 6](https://github.com/vladrabbit/K8S-2/blob/main/img/6.png)


### Решение 2.

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-wait-svc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-wait-svc
  template:
    metadata:
      labels:
        app: nginx-wait-svc
    spec:
      initContainers:
        - name: wait-for-svc
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Waiting for DNS record of Service 'nginx-svc'..."
              until nslookup nginx-svc.default.svc.cluster.local >/dev/null 2>&1; do
                echo "Service DNS not found yet, sleeping..."
                sleep 2
              done
              echo "Service DNS found. Continue."
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10

```

2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.

![Scr 1](https://github.com/vladrabbit/K8S-2/blob/main/img/2.1.png)



3. Создать и запустить Service. Убедиться, что Init запустился.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx-wait-svc
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP

```
![Scr 3](https://github.com/vladrabbit/K8S-2/blob/main/img/2.2.png)

4. Продемонстрировать состояние пода до и после запуска сервиса.

![Scr 3.1](https://github.com/vladrabbit/K8S-2/blob/main/img/2.2-1.png)

![Scr 3.2](https://github.com/vladrabbit/K8S-2/blob/main/img/2.3.png)

![Scr 3.3](https://github.com/vladrabbit/K8S-2/blob/main/img/2.3.1.png)





