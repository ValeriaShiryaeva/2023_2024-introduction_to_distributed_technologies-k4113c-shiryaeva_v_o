University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)  
Year: 2023/2024  
Group: K4113c  
Author: Shiryaeva Valeria  
Lab: Lab2  
Date of create: 17.10.2023  
Date of finished:
---
# Развертывание веб-сервиса в Minikube, доступ к веб-интерфейсу сервиса. Мониторинг сервиса
## Цель работы
Ознакомиться с типами "контроллеров" развертывания контейнеров, ознакомится с сетевыми сервисами и развернуть свое веб приложение.
## Ход работы
### 1. Создание deployment с 2 репликами контейнера
Создан файл `manifest.yaml`, который описывает конфигурацю deployment.

![manifest](/lab2/screenshots/manifest.jpg)

Разберем каждое поле манифеста:
- `apiVersion`: определяет версию API, используемую в манифесте.
- `kind`: определяет тип объекта, который создается с помощью этого манифеста.
- `metadata`: содержит метаданные Deployment:
  - `name`: определяет имя Deployment.
- `spec`: определяет спецификацию Deployment:
  - `replicas`: определяет количество реплик (Pod), которые будут запущены внутри Deployment.
  - `selector`: определяет какие поды должны быть управлемыми этим Deployment, основывая на метке `app: dep2`.
  - `template`: cодержит шаблон для создания подов, управляемых Deployment, включает метаданные и спецификацию контейнеров.
      - `metadata`: содержит метаданные для создаваемых подов, включая метку `app: dep2`.
      - `spec`: определяет спецификацию контейнеров в Pod:
          - `containers`: определяет список контейнеров, которые будут запущены внутри Pod.
            - `name`: имя контейнера, используется для идентификации контейнера внутри Pod.
            - `image`: образ контейнера, который будет запущен.
            - `ports`: определяет порты, которые будут прокинуты из контейнера для внутренних и внешних соединений.
                - `containerPort`: номер порта, который контейнер будет слушать внутри Pod.
                - `name`: имя порта, используется для идентификации порта внутри Pod и при настройке служб и других элементов.
            - `env`: определяет переменные окружения, передаваемые контейнеру. Устанавливаются переменные `REACT_APP_USERNAME` и `REACT_APP_COMPANY_NAME` с соответствующими значениями.

### 2. Создание Deployment
Создан Deployment в кластере с помощью команды:
```
kubectl apply -f manifest.yaml
```

![deployment_create](/lab2/screenshots/deployment_create.jpg)

### 3. Создание сервиса
Создан сервис в кластере с помощью команды:
```
kubectl expose deployment.apps dep2 —type=NodePort —port=3000
```

![service_create](/lab2/screenshots/service_create.jpg)

### 4. Проверка создания Deployment и Pods
Получим Deployment с помощью команды:
```
kubectl get deployments
```

![deployment_get](/lab2/screenshots/deployment_get.jpg)

Получим Pod с помощью команды:
```
 kubectl get pods
```

![pods_get](/lab2/screenshots/pods_get.jpg)

### 5. Прокидывание сервиса на порт
Прокинут порт локального компьютера в контейнер с помощью команды:
```
kubectl port-forward service/dep2 3000:3000
```

![connection](/lab2/screenshots/connection.jpg)


