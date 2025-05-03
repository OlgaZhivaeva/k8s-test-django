# Деплой в dev окружение в Яндекс Облаке
[ссылка на работающий сайт](https://edu-naughty-cray.sirius-k8s.dvmn.org/ )

## Подключитесь к кластеру в Yandex Claud

Установите интерфейс командной строки [Yandex Cloud CLI](https://yandex.cloud/ru/docs/cli/quickstart#install)

Добавьте учетные данные кластера Kubernetes в конфигурационный файл kubectl:

```
yc managed-kubernetes cluster get-credentials --id ID --external
```

Используйте утилиту kubectl для работы с кластером Kubernetes:

```
kubectl get pods
```

## Разместите образ Django-Site в docker registry

Перейдите в директорию `backend_main_django` в которой находится Dockerfile
 и выполните команды
```shell
$GIT_COMMIT_HASH = git rev-parse --short HEAD
docker build -t <docker_registry_login>/django-site:$GIT_COMMIT_HASH .
docker login
docker push <docker_registry_login>/django-site:$GIT_COMMIT_HASH 
```
## Создайте Secrets в кластере 

Для джанго приложения создайте в кластере файл Secret 
```shell
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
  namespace: edu-naughty-cray
type: Opaque
data:
  DEBUG: TRUE или FALSE
  SECRET_KEY: Секретный кдюч приложения
  ALLOWED_HOSTS: Список хостов/доменов
```
Значения переменных окружения необходимо закодировать в base64.
Пример кодирования
```shell
echo -n "FALSE" | base64
```
Проверьте наличие других секретов при помощи команды
```shell
kubectl get secrets -n edu-naughty-cray
```
Просмотрите содержимое секретов при помощи команд
```shell
kubectl get secret postgres -n edu-naughty-cray -o yaml
kubectl get secret pg-root-cert -n edu-naughty-cray -o yaml
```
Раскодировать значения переменных из секретов окружения можно при помощи команды
```shell
echo  "FALSE" | base64 -в
```
## Разверните приложение в кластере

Перейдите в директорию, где находятся файлы с манифестами `k8s-test-django/deploy/yc-sirius/edu-gaugty-cray` и выполните команды
```shell
kubectl apply -f secrets.yaml -n edu-gaugty-cray
kubectl apply -f deployment.yaml -n edu-gaugty-cray
kubectl apply -f service.yaml -n edu-gaugty-cray
```