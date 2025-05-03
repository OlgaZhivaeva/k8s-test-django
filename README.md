# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Разверните приложение в Minikube

Установите [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download) и [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)

Запустите кластер minikrube командой
```shell
minikube start
```
### Загрузите образ Джанго приложения в кластер

Для этого перейдите в директорию в которой находится DockerFile `k8s-test-django/backend_main_django` и выполните команду
```shell
minikube image build -t django_app .
```

### Создание Secrets в кластере

В директории `minikube` создайте файл `secrets.yaml` 

```
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
data:
  DATABASE_URL: Ссылка на базу данных
  DEBUG: TRUE или FALSE
  SECRET_KEY: Секретный кдюч приложения
  ALLOWED_HOSTS: Список хостов/доменов
```
Значения переменных окружения необходимо закодировать в base64.
Пример кодирования
```shell
echo -n "FALSE" | base64
```
Раскодировать можно командой
```shell
echo RkFMU0U= | base64 -d
```

### Разверните приложение в кластере

Перейдите в директорию где находятся файлы с манифестами `k8s-test-django/minikube` и выполните команды
```shell
kubectl apply -f secrets.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### Настройте Ingress

Подключите Ingress контроллер Nginx
```shell
minikube addons enable ingress
```
Создайте службу ingress командой
```shell
kubectl apply -f ingress.yaml
```

Получите IP адрес кластера minikube выполнив команду
```shell
minikube ip
```
Отредактируйте файл /etc/hosts на вашем компьютере, добавив в конец строку
<IP адрес кластера minikube> star-burger.test

### Запустите регулярную задачу для удаления сессий джанго

```shell
kubectl apply -f django-clearsessions.yaml
```
Сессии будут удаляться первого числа каждого месяца.

### Запустите задачу для применения миграций базы данных

```shell
kubectl apply -f django-migrate.yaml
```

## Создайте базу данных postgresql внутри кластера

### Установите [Helm](https://helm.sh/)


``` shell   
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Установте приложение PostgreSQL в ваш Kubernetes кластер
```shell
helm install my-postgres bitnami/postgresql \
  --set postgresqlUsername=test_k8s \
  --set postgresqlPassword=OwOtBep9Frut \
  --set postgresqlDatabase=test_k8s
```
