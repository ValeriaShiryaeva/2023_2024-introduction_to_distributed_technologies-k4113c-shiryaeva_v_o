University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)  
Year: 2023/2024  
Group: K4113c  
Author: Shiryaeva Valeria  
Lab: Lab1  
Date of create: 08.10.2023  
Date of finished:
---
# Установка Docker и Minikube, мой первый манифест
## Цель работы
Ознакомиться с инструментами Minikube и Docker, развернуть свой первый "под".
## Ход работы
### 1. Установка Docker, проверка текущей версии
Установлен Docker. На сриншоте представлена текущая версия Docker.

![docker_version](/lab1/screenshots/docker_version.jpg)

### 2. Установка Minikube, проверка текущей версии
Установлен Minikube. На скриншоте представлена текущая версия Minikube.

![minikube_version](/lab1/screenshots/minikube_version.jpg)

### 3. Запуск Minikube
Успешно запущен minikube cluster командой: 
```
minikube start
```

![start_minikube](/lab1/screenshots/start_minikube.jpg)

### 4. Установка kubectl, проверка текущей версии
Установлен kubectl. На скриншоте представлена текущая версия kubectl.

![kubectl_version](/lab1/screenshots/kubectl_version.jpg)

### 5. Создание манифеста
Создан файл `vault-pod.yaml`, который описывает конфигурацю "пода". На скриншоте представлена конфигурация "пода".

![vault-pod](/lab1/screenshots/vault-pod.jpg)

Разберем каждое поле манифеста:
- `apiVersion`: определяет версию API, используемую в манифесте.
- `kind`: определяет тип объекта, который создается с помощью этого манифеста. В данном случае, создается одиночный контейнер.
- `metadata`: содержит метаданные для Pod:
  - `name`: определяет имя Pod.
  - `labels`: позволяет присвоить ключ-значение метки Pod. Метка `app: vault` может быть использована для поиска и фильтрации Pods.
- `spec`: определяет спецификацию Pod:
  - `containers`: определяет список контейнеров, которые будут запущены внутри Pod. В данном случае, один контейнер с именем `vault` и образом `vault:1.13.3`.
    - `name`: имя контейнера, используется для идентификации контейнера внутри Pod.
    - `image`: образ контейнера, который будет запущен.
    - `ports`: определяет порты, которые будут прокинуты из контейнера. В данном случае один порт с именем `http_pod_8200` и номером `8200`, который будет доступен для внутренних и внешних соединений.
      - `containerPort`: номер порта, который контейнер будет слушать внутри Pod.
      - `name`: имя порта, используется для идентификации порта внутри Pod и при настройке служб и других элементов.
### 6. Создание "пода" в кластере
Создан "под" в кластере с помощью команды:
```
kubectl apply -f vault-pod.yaml
```
Результат выполнения команды:

![vault_create](/lab1/screenshots/vault_create.jpg)

### 7. Cоздание сервиса для доступа к контейнеру
Для доступа к контейнеру создан сервис с помощью команды:
```
minikube kubectl -- expose pod vault --type=NodePort --port=8200
```
Результат выполнения команды:

![service_create](/lab1/screenshots/service_create.jpg)

### 8. Прокидывание сервиса на порт
Для того, чтобы зайти в контейнер, прокинут порт локального компьютера в контейнер с помощью команды:
```
minikube kubectl -- port-forward service/vault 8200:8200
```
Результат выполнения команды:

![connection](/lab1/screenshots/connection.jpg)

После этого можно зайти по ссылке [http://localhost:8200](http://localhost:8200), где откроется интерфейс.

![interface_vault](/lab1/screenshots/interface_vault.jpg)

### 9. Получение токена для входа в Vault
Для того, чтобы найти токен, открыты логи с помощью комнады:
```
kubectl logs vault
```
Результат выполнения команды:

![token](/lab1/screenshots/token.jpg)

Токен для входа располагается в поле `Root Token`. С помощью него можно авторизоваться в Vault, что показано на скрине.

![log_vault](/lab1/screenshots/log_vault.jpg)

## Ответы на вопросы

1. `kubectl apply -f vault-pod.yaml` создает или обновляет ресурсы (объекты) в кластере на основе конфигурационного файла. Флаг `-f` указывает имя файла или URL-адрес файла, который содержит описание ресурсов.
2. `minikube kubectl -- expose pod vault --type=NodePort --port=8200` создает службу из указанного объекта с заданным именем. Флаг `--type=NodePort` указывает тип службы. Флаг `--port=8200` указывает порт, который открыт для доступа к создаваемой службе.
3. `minikube kubectl -- port-forward service/vault 8200:8200` настраивает прямое соединение между локальной машиной и ресуорсом внутри кластера. Агумент команды `service/vault` указывает на установление прямого соединения с сервисом Vault внутри кластера. Аргумент `8200:8200`определяет порты для прокидывания.

## Схема организации контейеров и сервисов

![scheme](/lab1/screenshots/scheme.jpg)

