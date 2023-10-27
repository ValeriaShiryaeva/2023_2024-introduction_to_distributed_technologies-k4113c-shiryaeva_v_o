
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
  name: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
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
  name: app
spec:
  type: ClusterIP
  ports:
    - port: 3000
      protocol: TCP
      targetPort: 3000
  selector:
    app: app
```

### 3. Генерация TLS сертификата
Для генерации TLS сертификата будем использовать инструмент командной строки `openssl`. 
Для того, чтобы сгенерировать сертификат нужно выполнить команду:
```
openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out cert.crt -keyout cert.key
```

![]()

Разберем параметры команды:
  - `- new`: создет новый запрос на сертификат.
  - `- newkey rsa:4096`: создает новый закрытый ключ RSA с длиной 4096 бит.
  - `- x509`: создает самоподписанный X.509 сертификат.
  - `- sha256`: определяет алгоритм хэширования, используемый для сертификата.
  - `- days 365`: устанавливает срок действия сертификата в днях.
  - `- nodes`: создает закрытый ключ без паролей (без шифрования).
  - `- out cert.crt`: имя файла, в который будет сохранен сертификат (cert.crt).
  - `- keyout cert.key`: имя файла, в который будет сохранен закрытый ключ (cert.key).

Далее создаем секрет, добавляя сертификат в `minikube` с помощью команды:
```
kubectl create secret tls valeria-lab3-tls —key="cert.key" —cert="cert.crt"
```

![]()

### 4. Создание Ingress 
Сначала нужно запустить `minikube` с помощью команды:
```
minikube start
```
Далее необходимо подключить `ingress` с помощью команды:
```
minikube addons enable ingress
```

![]()

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
    - `hosts`: cписок хостов, для которых действует TLS.
    - `secretName`: название секрета Kubernetes, который содержит TLS-сертификат и закрытый ключ.
  - `rules`: определяет правила маршрутизации в Ingress:
    - `host`: задает хост, для которого применяется это правило.
    - `http`: определяет правила HTTP-маршрутизации:
      - `paths`: определяет пути запросов HTTP:
        - `path`: задает путь запросов.
          - `pathType`: eказывает тип пути как "Prefix", то есть запросы, начинающиеся с /, будут соответствовать этому пути.
          - `backend`: определяет службу, на которую должны быть направлены запросы:
            - `service`: задает имя службы, к которой будут направлены запросы:
              - `name`: имя службы
              - `port`: определяет номер порта службы, на который будут направлены запросы .

### 5. Запуск написаного манифеста содержащего ConfigMap, Deployment, Service, Ingress
Для примененения всего написанного применим конфигурации из написанного манифеста с помощью команды:
```
kubectl apply -f manifest.yaml
```

![]()


### 6. Переход в браузер
Для т

### 7. Проверка наличия сертификата

## Схема