# zend0_platform

# Установка kubectl
Установим последнюю доступную версию kubectl на локальнуюмашину. 
Инструкции по установке доступны по [ссылке](https://kubernetes.io/docs/tasks/tools/install-kubectl/).  
Настройка [автодополнения](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete) для shell.

# minicube
## Установка minikube
Инструкции по установке доступны по [ссылке](https://kubernetes.io/docs/tasks/tools/install-minikube/).  

Указываем KVM как единственный драйвер для minikube
```shell script
minikube config set vm-driver kvm2
```
## Работа с minicube
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
**kubectl run** - запустить ресурс
**frontend** - с именем frontend
**--image** - из образа bbilder/otus-hipster-frontend
**--restart=Neverу** указываем на то, что в качестве ресурса запускаем 
pod. [Подробности](https://kubernetes.io/docs/reference/kubectl/conventions/)  

Один из распространенных кейсов использования ad-hoc режима - генерация 
манифестов средствами kubectl:
```shell script
kubectl run frontend --image bbilder/otus-hipster-frontend --restart=Never --dry-run -o yaml > frontend-pod.yaml
```
Рассмотрим дополнительные ключи:  
**--dry-run** - вывод информации о ресурсе без его реального создания
**-o yaml** - форматирование вывода в YAML

## Проверка работы приложения
Проверим работоспособность web сервера. Существует несколько способов получить 
доступ к pod, запущенным внутри кластера.  
Воспользуемся командой [kubectl port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
```shell script
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
```
В качестве альтернативы `kubectl port-forward` можно использовать удобную обертку [kube-forwarde](https://kube-forwarder.pixelpoint.io/).


