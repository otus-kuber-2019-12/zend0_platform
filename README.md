# zend0_platform

# Установка kubectl

Установим последнюю доступную версию kubectl на локальнуюмашину. 
Инструкции по установке доступны по [ссылке](https://kubernetes.io/docs/tasks/tools/install-kubectl/).  
Настройка [автодополнения](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete) для shell.

# minikube

## Установка minikube

Инструкции по установке доступны по [ссылке](https://kubernetes.io/docs/tasks/tools/install-minikube/).  

Указываем KVM как единственный драйвер для minikube

```shell script
minikube config set vm-driver kvm2
```

## Работа с minikube

Запуск

```shell script
minikube start
```

подключение к VM

```shell script
minicube ssh
```

Для визуализации и управления, есть [dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
или консольная утилита [k9s](https://github.com/derailed/k9s)

## Манифест pod

применение манифеста

```shell script
kubectl apply -f web-pod.yaml
```

получение манифеста для уже запущенно пода

```shell script
kubectl get pod web -o yaml
```

Другой способ посмотреть описание pod-использовать ключ `describe`.
Команда позволяет отследить текущее состояние объекта, а также события, которые с ним 
происходили

```shell script
kubectl describe pod web
```

## Создание pod в режиме ad-hoc

на примере запуска **frontend** pod

```shell script
kubectl run frontend --image bbilder/otus-hipster-frontend --restart=Never
```

* **kubectl run** - запустить ресурс
* **frontend** - с именем frontend
* **--image** - из образа bbilder/otus-hipster-frontend
* **--restart=Neverу** указываем на то, что в качестве ресурса запускаем 
pod. [Подробности](https://kubernetes.io/docs/reference/kubectl/conventions/)  

Один из распространенных кейсов использования ad-hoc режима - генерация 
манифестов средствами kubectl:

```shell script
kubectl run frontend --image bbilder/otus-hipster-frontend --restart=Never --dry-run -o yaml > frontend-pod.yaml
```

Рассмотрим дополнительные ключи:  

* **--dry-run** - вывод информации о ресурсе без его реального создания
* **-o yaml** - форматирование вывода в YAML

## Проверка работы приложения

Проверим работоспособность web сервера. Существует несколько способов получить 
доступ к pod, запущенным внутри кластера.  
Воспользуемся командой [kubectl port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)

```shell script
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
```

В качестве альтернативы `kubectl port-forward` можно использовать удобную обертку [kube-forwarde](https://kube-forwarder.pixelpoint.io/).

# kind

## Установка kind

Инструкции по установке доступны по [ссылке](https://kind.sigs.k8s.io/docs/user/quick-start).  

## Работа с kind

Запускаем создание кластера kind (`kind-config.yaml` - описание кластера)

```shell script
kind create cluster --config kind-config.yaml
```

Удаление кластера

```bash
kind delete cluster
```

# Контроллеры

## ReplicationController и ReplicaSet

* Следит за тем, чтобы число подов соответствовало заданному
  * Умеет пересоздавать Podы при отказе узла (обычные "голые" поды умирают вместе с нодой)
  * Умеет добавлять/удалять Podы не пересоздавая всю группу
* **НЕ** проверяет соответствие запущенных Podов шаблону

_**Важно**, ReplicaSet и ReplicationController не умеют рестартовать запущенные поды при обновлении шаблона, они только следят за их количеством.
То есть при обновлении версии образа в манифесте, поды не будут рестартованны._  

Увеличиваем количество реплик сервиса ad-hoc командой

```shell script
kubectl scale replicaset <pod_name> --replicas=3
```

Проверям, что ReplicaSet контроллер теперь управляет тремя репликами

```shell script
$ kubectl get rs <pod_name>
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       5m45s
```

проверим образ, указанный в ReplicaSet (о форматировании вывода можно почитать в [документации](https://kubernetes.io/docs/reference/kubectl/jsonpath/))

```shell script
kubectl get replicaset frontend -o=jsonpath='{.spec.template.spec.containers[0].image}'
```

И образ из которого сейчас запущены pod, управляемые контроллером

```shell script
kubectl get pods -l app=frontend -o=jsonpath='{.items[0:3].spec.containers[0].image}'
```

## Deployment

* Это контроллер контроллеров, управляющий ReplicaSets
* Это "декларативный" способ развертывания
* И рекомендованный способ запуска Podов (даже если нужна только одна реплика)

[Стратегии](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy):  
blue-green:

* Развертывание трех новых pod
* Удаление трех старых pod

Reverse Rolling Update:

* Удаление одного старого pod
* Создание одного нового pod

Вывод deployment

```shell script
kubectl get deployments
```

просмотр истории версий нашего Deployment

```shell script
kubectl rollout history deployment <pod_name>
```

Делаем откат на предыдущую версию deployment

```shell script
kubectl rollout undo deployment paymentservice --to-revision=1 | kubectl get rs -lapp=paymentservice -w
```

## Probes

Проверка подов на готовность / доступность
[Документация](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)  

Для отслеживания статуса деплоя в CI/CD можно воспользоваться командой

```shell script
kubectl rollout status deployment/frontend
```

## DaemonSet

Ещё один контроллер, отличительная особенность DaemonSet в том, что при его применении на каждом физическом хосте 
создается по одному экземпляру pod, описанного в спецификации.  

Типичные кейсы использования DaemonSet:

* Сетевые плагины
* Утилиты для сбора и отправки логов (Fluent Bit, Fluentd, etc...)
* Различные утилиты для мониторинга (Node Exporter, etc...)

_В рамках ДЗ, где разбираемся с daemonset и node-exporter, перед применением node-exporter-daemonset.yaml надо обратить 
и на [это](https://github.com/coreos/kube-prometheus/blob/master/manifests/node-exporter-serviceAccount.yaml)_

## Права доступа, роли, RBAC и вот это всё

* Аккаунты пользователей - для пользователей,то есть для людей 
  * Service Account для процессов в Pod
* Аккаунты пользователей глобальны в рамках кластера
  * Service Accounts живут в рамках namespace (это удобно для разных приложений в нутри кластера)

Существует 4 модуля авторизации:

* Node
* ABAC
* RBAC
* Webhook

Узнать что за авторизация настроенна в кластере:

```bash
kubectl cluster-info dump | grep authorization-mode
```

## Сетевое взаимодействие Pod, сервисы

[Kubernetes Services and Iptables](https://msazure.club/kubernetes-services-and-iptables/)  

### Создание Service || ClusterIP

`ClusterIP` удобны в тех случаях, когда:

* Нам не надо подключаться к конкретному поду сервиса
* Нас устраивается случайное расределение подключений междуподами
* Нам нужна стабильная точка подключения к сервису,независимая от подов, нод и DNS-имен

Например:

* Подключения клиентов к кластеру БД (multi-read) или хранилищу
* Простейшая (не совсем, use IPVS, Luke) балансировка нагрузкивнутри кластера

[Описание работы и настройки IPVS в K8s](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md)

#### Как посмотреть конфигурацию IPVS (в minikube)

Ведь в ВМ нет утилиты `ipvsadm`?  
В ВМ выполним команду `toolbox` - в результате мы окажемся в контейнере с Fedora  
Теперь установим `ipvsadm`:

```bash
dnf install -y ipvsadm && dnf clean all
```

## Volumes, Storages, Stateful приложения

Проверка ресурсов

```bash
kubectl get statefulsets
kubectl get pvc
kubectl get pv
kubectl describe <resource> <resource_name>
```

## Работа с секретами

[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

Максимальный размер секрета: 1 МБ

Проверка секретов:

```bash
kubectl get secrets
```

Вывод по больше информации о секрете

```bash
kubectl describe secrets/db-user-pass
```

Полный вывод секрета

```bash
kubectl get secret mysecret -o yaml
```

Кодирование / декодирование секрета

```bash
echo -n 'admin' | base64
echo 'YWRtaW4=' | base64 --decode
```
