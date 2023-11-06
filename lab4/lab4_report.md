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

Для проверки работы CNI Calico, посмотрим поды с меткой `calico-node` с помощтю команды:
```
kubectl get pods -l k8s-app=calico-node -A
```
![get_pods](/lab4/screenshots/get_pods.jpg)
