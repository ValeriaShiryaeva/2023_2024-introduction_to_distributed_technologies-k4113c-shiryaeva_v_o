University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)  
Year: 2023/2024  
Group: K4113c  
Author: Shiryaeva Valeria  
Lab: Lab4  
Date of create: 06.11.2023  
Date of finished: 
---
# Сети связи в Minikube, CNI и CoreDNS
## Цель работы
Познакомиться с CNI Calico и функцией IPAM Plugin, изучить особенности работы CNI и CoreDNS.
## Ход работы
### 1. Установка плагина и режима работы
При запуске `minikube` необходимо установить плагин `CNI=calico` и режим работы  `Multi-Node Clusters` с помощью команды, которая разворачивает 2 ноды:
```
minikube start --network-plugin=cni --cni=calico --nodes 2 -p multinode-demo
```

![minikube_start](/lab4/screenshots/minikube_start.jpg)

Рассмотрим опции, которые указаны в команде:
  - `--network-plugin=cni`: указывает Minikube использовать плагин сети типа Container Network Interface (CNI). CNI - это стандартный интерфейс для настройки сетей в контейнерах (в данном случае, для настройки сети в кластере).
  - `--cni=calico`: указывает конкретно на использование CNI-плагина Calico.
  - `--nodes 2`: определяет количество узлов (нод) в кластере.
  - `-p multinode-demo`: задает профиль Minikube, который позволяет именовать и сохранять конфигурацию для будущего использования.

### 2. Проверка работы плагина и количества нод
Для проверки количества нод можно использовать команду:
```
kubectl get nodes
```
![get_nodes](/lab4/screenshots/get_nodes.jpg)

Для проверки работы CNI Calico, посмотрим поды с меткой `calico-node` с помощью команды:
```
kubectl get pods -l k8s-app=calico-node -A
```
![get_pods](/lab4/screenshots/get_pods.jpg)

### 3. Пометка нод по принципу стойки
Для проверки режима `IPAM` необходимо для запущеных нод указать `label` по признаку стойки с помощью команды:
```
kubectl label nodes <node-name> rack=<rack-id>
```
Где указывается название ноды и его id, в моем случае выполняем такие команды:
```
kubectl label nodes multinode-demo rack=0
kubectl label nodes multinode-demo-m02 rack=1
```

![label_nodes](/lab4/screenshots/label_nodes.jpg)

![label_nodes2](/lab4/screenshots/label_nodes2.jpg)

### 4. Создание манифеста для Calico
Для назначения IP адресов в Calico необходимо написать манифест для IPPool ресурса. С помощью IPPool можно создать IP-pool (блок IP-адресов), который выделяет IP-адреса только для узлов с меткой, которую мы указали ранее.
```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: rack-0
spec:
  cidr: 192.168.10.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: rack == "0"

---

apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: rack-1
spec:
  cidr: 192.168.20.0/24
  ipipMode: Always
  natOutgoing: true
  nodeSelector: rack == "1"
```
Рассмотрим поля манифеста:
- `apiVersion`: версия API, используемая для API Calico.
- `kind`: определяет тип объекта, который создается с помощью этого манифеста.
- `metadata`: содержит метаданные IPPool:
  - `name`: определяет имя IPPool, по которому его можно будет идентифицировать внутри кластера.
- `spec`: определяет спецификацию IPPool:
  - `cidr`: определяет диапазон IP-адресов в формате CIDR (Classless Inter-Domain Routing), который будет предоставлен данным пулом.
  - `ipipMode`: определяет режим использования протокола IP-in-IP (IPIP).
  - `natOutgoing`: определяет, будет ли выполняться Network Address Translation (NAT) при исходящем сетевом трафике из пула IP-адресов.
  - `nodeSelector`: определяет, к каким узлам (нодам) применяется данный пул IP-адресов.

 Чтобы применить манифест для IPPool, надо установить calicoct, для этого добавим в домашнюю директорию добавим файл `calicoctl.yaml`, используя команду:
```
kubectl create -f calicoctl.yaml
```
![create_calicoctl](/lab4/screenshots/create_calicoctl.jpg)

После чего необоходимо удалить IPPool по-умолчанию с помощью команды:
```
kubectl delete ippools default-ipv4-ippool
```

![delete_ippools_default](/lab4/screenshots/delete_ippools_default.jpg)

Далее создаем пулы с помощью команды:
```
kubectl exec -i -n kube-system calicoctl -- /calicoctl --allow-version-mismatch create -f - < manifest.yaml
```

![create_manifest](/lab4/screenshots/create_manifest.jpg)

Проверяем, что создалось два пула:
```
kubectl exec -i -n kube-system calicoctl -- /calicoctl --allow-version-mismatch get ippool -o wide
```

![get_ippool](/lab4/screenshots/get_ippool.jpg)

### 5. Создание манифеста содержащего ConfigMap, Deployment, Service
Мы можем использовать уже написанный манифест в прошлой лабораторной работе. Единственное, нужно поменять значение в секции `type` на `LoadBalancer`.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: lab3
spec:
  ports:
    - port: 3000
      protocol: TCP
      targetPort: 3000
  selector:
    app: lab3
  type: LoadBalancer
```
После чего можно запустить написанный манифест, содержащий ConfigMap, Deployment, Service, с помощью комнады:
```
kubectl apply -f manifest.yaml
```

![create_manifest1](/lab4/screenshots/create_manifest1.jpg)

Проверим IP созданных подов с помощью команды:
```
kubectl get pods -o wide
```

![get_pods1](/lab4/screenshots/get_pods1.jpg)

### 6. Проброс портов и проверка переменных
Необходимо пробросить порт для подключения к сервису через браузер с помощью комнады:
```
kubectl port-forward service/lab3 3000:3000
```
Далее по адресу открываем ссылку: `127.0.0.1:3000`

![lab3_pods](/lab4/screenshots/lab3_pods.jpg)

Также до второго пода можно добраться, пробросив порт на на сам под с помощью команды:
```
kubectl port-forward pod/lab3-bf47574b7-6qzjc 3000:3000
```

![port-forward](/lab4/screenshots/port-forward.jpg)

Далее по адресу открываем ссылку: `127.0.0.1:3000`

![lab3_pods1](/lab4/screenshots/lab3_pods1.jpg)

### 7. Проверка переменных
Как можно заметить переменные `REACT_APP_USERNAME` и `REACT_APP_COMPANY_NAME` на скриншоте соответствуют переменным в `ConfigMap`. 
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
Перемнные `Container name` и `Container IP` меняются от контейнера к контейнеру, потому что задаются для каждого отдельно.

### 8. Пинг соседа
Снячала нужно `FQDN`пода, для этого используем команду
```
kubectl exec lab3-bf47574b7-6qzjc -- nslookup 192.168.20.193
```

![nslookup_193](/lab4/screenshots/nslookup_193.jpg)

По найденному имени в поле `name` можно пигновать:
```
kubectl exec lab3-bf47574b7-6qzjc -- ping 192-168-20-193.lab3.default.svc.cluster.local
```

![ping_193](/lab4/screenshots/ping_193.jpg)

Можно заметить, что пакеты успешно доходят до адресата и обратно.

### 8.1 Второй способ пинг
Пингуем второй контейнер `lab3-bf47574b7-6qzjc` с IP-адресом: `ping 192.168.20.193` с помощью команды:
```
kubectl exec -ti lab3-bf47574b7-6qzjc -- sh
```

![ping_193_2](/lab4/screenshots/ping_193_2.jpg)

Можно заметить, что пакеты успешно доходят до адресата и обратно.

## Схема

![scheme](/lab4/screenshots/scheme.jpg)
