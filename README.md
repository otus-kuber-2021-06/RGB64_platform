# RGB64_platform
RGB64 Platform repository

## Домашняя работа kubernetes-intro

### Задание 1

Разберитесь почему все pod в namespace kube-system восстановились после удаления. Укажите причину в описании PR

### Решение

Подом **coredns** управляет объект **coredns-74ff55c5b** типа ReplicaSet:

```
ruslan@kube-study:~/Projects/RGB64_platform/RGB64_platform$ kubectl get rs -n kube-system
NAME                DESIRED   CURRENT   READY   AGE
coredns-74ff55c5b   1         1         1       8d
```

ReplicaSet создан объектом **coredns** типа Deployment:

```
ruslan@kube-study:~/Projects/RGB64_platform/RGB64_platform$ kubectl get deployments -n kube-system
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   1/1     1            1           8d
```

У ReplicaSet есть аннотация `deployment.kubernetes.io/desired-replicas: "1"`:

```
ruslan@kube-study:~/Projects/RGB64_platform/RGB64_platform$ kubectl get rs coredns-74ff55c5b -n kube-system -o yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  annotations:
    deployment.kubernetes.io/desired-replicas: "1"
    deployment.kubernetes.io/max-replicas: "2"
    deployment.kubernetes.io/revision: "1"
...
```

И в описании Deployment в строке Replicas `1 desired`:

```
ruslan@kube-study:~/Projects/RGB64_platform/RGB64_platform$ kubectl describe deployment coredns -n kube-system
Name:                   coredns
Namespace:              kube-system
CreationTimestamp:      Thu, 15 Apr 2021 22:43:39 +0400
Labels:                 k8s-app=kube-dns
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               k8s-app=kube-dns
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
...
```

Итого - под **coredns** перезапускается, потому что Deployment требует наличия одной реплики и делегирует заниматься этим ReplicaSet-у. 

Подом **kube-proxy** управляет объект **kube-proxy** типа DaemonSet:

```
ruslan@kube-study:~/Projects/RGB64_platform/RGB64_platform$ kubectl get daemonsets -n kube-system
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   9d

```

Остальные поды являются статическими и их перезапускает kubelet.


### Задание 2

#### Создание образа контейнера

Создан простейший html-файл homework.html.

Создан файл настроек web-сервера nginx, содержащий инструкции слушать порт 8000 и использовать папку /app в качестве корневой:

```
server {
 listen 8000;
 location / {
   alias /app/;
 }
}
```

Создан Dockerfile, описывающий образ вот так:
* За основу берем образ nginx последней версии
* Создаем каталог /app
* В каталог /app копируем файл homework.html
* Создаем файл идентификатора процесса nginx - nginx.pid
* Назначаем пользователя с uid 1001 владельцем папки /app, pid-файла, а также папок с настройками, кэшем и логами nginx
* Задаем пользователя с uid 1001 для использования во всех последующих инструкциях
* Копируем в образ файл настроек default.conf
* Запускаем nginx с глобальной директивой конфигурации `daemon off` (чтобы nginx оставался на переднем плане, для целей отладки)
* Выставляем порт 8000

```
FROM nginx:latest

RUN mkdir /app

COPY homework.html /app

RUN touch /var/run/nginx.pid

RUN chown -R 1001 /app /etc/nginx /var/cache/nginx /var/log/nginx /var/run/nginx.pid

USER 1001

COPY default.conf /etc/nginx/conf.d/default.conf

CMD ["nginx", "-g", "daemon off;"]

EXPOSE 8000
```

Образ контейнера собран и помещен в публичный реестр контейнеров Docker Hub (https://hub.docker.com/r/rgb64/kubernetesintro).


#### Создание pod

Написан манифест `web-pod.yaml` для создания pod **web**, содержащего один контейнер, созданный на предыдущем шаге. Pod запущен с помощью `kubectl apply -f web-pod.yaml` 
Далее в манифесте был указан несуществующий тэг образа и манифест применен заново, статус pod изменился, т.к. он не смог успешно запуститься.
Затем в манифест было добавлено описание init контейнера, генерирующего страницу `index.html` с помощью скрипта с гитхаба express42.
Для того, чтобы файл, созданный в init контейнере, был доступен в основном контейнере, использован **volume** типа `emptyDir`.
Запущенный pod **web** удален из кластера и создан новый. С помощью `kubectl port-forward` и браузера открыта страница index.html.

Итоговый манифест:

```
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  volumes:
  - name: app
    emptyDir: {}
  containers:
  - name: web
    image: rgb64/kubernetesintro
    volumeMounts:
    - name: app
      mountPath: /app
  initContainers:
  - name: init
    image: busybox:1.31.0
    command: ['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh']
    volumeMounts:
    - name: app
      mountPath: /app
```

### Задание со звездочкой

Склонирован репозиторий `https://github.com/GoogleCloudPlatform/microservices-demo`, собран образ для микросервиса `frontend`, образ помещен в Docker Hub (https://hub.docker.com/r/rgb64/hipster-frontend).
С помощью команды `kubectl run frontend --image rgb64/hipster-frontend:v0.0.1 --restart=Never` запущен pod **frontend**, но он находится в статусе `Error`:

```
ruslan@kube-study:~/Projects/RGB64_platform/RGB64_platform/kubernetes-intro$ kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
frontend   0/1     Error     0          42s
web        1/1     Running   3          4d20h
```

В логах pod **frontend** видно, что не установлено значение переменной окружения `"PRODUCT_CATALOG_SERVICE_ADDR"`:

```
ruslan@kube-study:~/Projects/RGB64_platform/RGB64_platform/kubernetes-intro$ kubectl logs frontend
{"message":"Tracing enabled.","severity":"info","timestamp":"2021-04-25T15:32:12.628813761Z"}
{"message":"Profiling enabled.","severity":"info","timestamp":"2021-04-25T15:32:12.62894772Z"}
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set
```
Значение этой переменной, а также имена и значения других переменных подсматриваем в манифесте в каталоге `kubernetes-manifests` репозитория `microservices-demo`. Итоговый манифест:

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: frontend
  name: frontend
spec:
  containers:
  - image: rgb64/hipster-frontend:v0.0.1
    name: frontend
    resources: {}
    env:
    - name: PRODUCT_CATALOG_SERVICE_ADDR
      value: "productcatalogservice:3550"
    - name: CURRENCY_SERVICE_ADDR
      value: "currencyservice:7000"
    - name: CART_SERVICE_ADDR
      value: "cartservice:7070"
    - name: RECOMMENDATION_SERVICE_ADDR
      value: "recommendationservice:8080"
    - name: SHIPPING_SERVICE_ADDR
      value: "shippingservice:50051"
    - name: CHECKOUT_SERVICE_ADDR
      value: "checkoutservice:5050"
    - name: AD_SERVICE_ADDR
      value: "adservice:9555"
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

```