
University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)  
Year: 2023/2024  
Group: K4113c  
Author: Shiryaeva Valeria  
Lab: Lab3  
Date of create: 26.10.2023  
Date of finished:
---
# Сертификаты и "секреты" в Minikube, безопасное хранение данных
## Цель работы
Познакомиться с сертификатами и "секретами" в Minikube, правилами безопасного хранения данных в Minikube.
## Ход работы
### 1. Cоздание configMap с переменными: REACT_APP_USERNAME, REACT_APP_COMPANY_NAME
`ConfigMap` - это объект API, используемый для хранения неконфиденциальных данных в парах ключ-значение. Он позволяют разделить данные конфигурации и код приложения.
Создаем манифест для `ConfigMap` с переменными `REACT_APP_USERNAME` и `REACT_APP_COMPANY_NAME` 

```yaml
apiVersion: v1
data:
  REACT_APP_USERNAME: "Shiryaeva Valeria"
  REACT_APP_COMPANY_NAME: "ITMO"
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: config
```

### 2. Создание Deployment с 2 репликами контейнера и Service
В прошлой лабораторной работе был уже написан шаблон для Deployment с 2 репликами контейнера. 
Единственное, необходимо заменить секцию `env` на `envForm`, чтобы переменные были взяты из `ConfigMap`, а не прописаны, как было раньше.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab3
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lab3
  template:
    metadata:
      labels:
        app: lab3
    spec:
      containers:
      - name: itdt-contained-frontend
        image: ifilyaninitmo/itdt-contained-frontend:master
        ports:
        - containerPort: 3000
        envFrom:
          - configMapRef:
              name: config
```

Также в прошлой лабораторной работе сервис создавался командой. Но в данной лабораторной работе удобнее прописать его создание в манифесте. 

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lab3
spec:
  type: ClusterIP
  ports:
    - port: 3000
      protocol: TCP
      targetPort: 3000
  selector:
    app: lab3
```

### 3. Генерация TLS сертификата
Для генерации TLS сертификата будем использовать инструмент командной строки `openssl`. 
Для того, чтобы сгенерировать сертификат нужно выполнить команду:
```
openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out cert.crt -keyout cert.key
```

![create_certificate](/lab3/screenshots/create_certificate.jpg)

Разберем параметры команды:
  - `- new`: создет новый запрос на сертификат.
  - `- newkey rsa:4096`: создает новый закрытый ключ RSA с длиной 4096 бит.
  - `- x509`: создает самоподписанный X.509 сертификат.
  - `- sha256`: определяет алгоритм хэширования, используемый для сертификата.
  - `- days 365`: устанавливает срок действия сертификата в днях.
  - `- nodes`: создает закрытый ключ без паролей (без шифрования).
  - `- out cert.crt`: имя файла, в который будет сохранен сертификат (cert.crt).
  - `- keyout cert.key`: имя файла, в который будет сохранен закрытый ключ (cert.key).

При заполнении инфрмации обязательным полем является `Common Name` указываем доменное имя (FQDN), по которому с помощью Ingress, мы будем заходить на сервер `valeria-lab3.app`.

После выполнения данной команды, создаются 2 файла с заданными именами, которые представлены на скриншоте:

![create_file](/lab3/screenshots/create_file.jpg)

Далее создаем секрет, добавляя сертификат в `minikube` с помощью команды:
```
kubectl create secret tls valeria-lab3-tls --key="cert.key" --cert="cert.crt"
```

![create_secret](/lab3/screenshots/create_secret.jpg)

### 4. Создание Ingress 
`Ingress` - это ресурс, с помощью которого мы можем задать единую точку входа в наш кластер. Он позволяет нам назначить для каждого сервиса свой URL, доступный вне кластера.

Чтобы подключить `ingress`, необходимо выполнить команду:
```
minikube addons enable ingress
```

![enable_ingress](/lab3/screenshots/enable_ingress.jpg)

Далее необходимо написать манифест для `ingress`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
spec:
  tls:
    - hosts:
        - valeria-lab3.app
      secretName: valeria-lab3-tls
  rules:
    - host: valeria-lab3.app
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 3000
```

Разберем поля этого манифеста:

- `apiVersion`: версия API, используемая для Ingress.
- `kind`: определяет тип объекта, который создается с помощью этого манифеста.
- `metadata`: содержит метаданные Ingress:
  - `name`: определяет имя Ingress, по которому его можно будет идентифицировать внутри кластера.
- `spec`: jпределяет спецификацию Ingress:
  - `tls`: определяет настройки TLS для Ingress:
    - `hosts`: cписок хостов, для которых действует TLS (указываем FQDN).
    - `secretName`: название секрета Kubernetes, который содержит TLS-сертификат и закрытый ключ.
  - `rules`: определяет правила маршрутизации в Ingress:
    - `host`: задает хост, для которого применяется это правило (указываем FQDN).
    - `http`: определяет правила HTTP-маршрутизации:
      - `paths`: определяет пути запросов HTTP:
        - `path`: задает путь запросов.
          - `pathType`: eказывает тип пути как "Prefix", то есть запросы, начинающиеся с /, будут соответствовать этому пути.
          - `backend`: определяет службу, на которую должны быть направлены запросы:
            - `service`: задает имя службы, к которой будут направлены запросы:
              - `name`: имя службы
              - `port`: определяет номер порта службы, на который будут направлены запросы .

### 5. Запуск написаного манифеста содержащего ConfigMap, Deployment, Service, Ingress
Для примененения всего написанного в манифесте выполним команду:
```
kubectl apply -f manifest.yaml
```

![create_manifest](/lab3/screenshots/create_manifest.jpg)

Теперь проверим с помощью команд то, что создалось после:
```
kubectl get configmaps
```

![get_configmaps](/lab3/screenshots/get_configmaps.jpg)

```
kubectl get deployments
```

![get_deployments](/lab3/screenshots/get_deployments.jpg)

```
kubectl get pods
```

![get_pods](/lab3/screenshots/get_pods.jpg)

```
kubectl get services
```

![get_services](/lab3/screenshots/get_services.jpg)

```
kubectl get ingress
```

![get_ingress](/lab3/screenshots/get_ingress.jpg)


### 6. Переход в браузер
Для начала необходимо добавить запись в файл `hosts`, чтобы связать ip-адрес и доменное имя (FQDN), для этого выполним комнаду:
```
echo "127.0.0.1 valeria-lab3.app" | sudo tee -a /etc/hosts
```
Данная команда выполняет вывод строки `127.0.0.1 valeria-lab3.app` в файл `/etc/hosts`.

![add_hosts](/lab3/screenshots/add_hosts.jpg)

Проверим, что в файл записалась строка `127.0.0.1 valeria-lab3.app`

![view_hosts](/lab3/screenshots/view_hosts.jpg)

Подключаемся к Ingress командой с помощью команды:
```
minikube tunnel
```

![start_tunnel](/lab3/screenshots/start_tunnel.jpg)

Открываем страницу с адресом  `https://valeria-lab3.app`

![open_site](/lab3/screenshots/open_site.jpg)

Но из-за самоподписанного сертификата браурез не позволяет зайти на сайте. В браузере Google Chrome можно написать команду `thisisunsafe` после чего мы сможем зайти в браузер (инструкция на сайте: https://www.usitility.com/google-chrome/how-to-fix-connection-is-not-private-chrome/).

![](/)

### 7. Проверка наличия сертификата
Ниже представленны данные сертивиката:

![view_certificate](/lab3/screenshots/view_certificate.jpg)

## Схема
