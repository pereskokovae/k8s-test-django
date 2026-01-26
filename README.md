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


## Развертывание в minikube
1. Запуск кластера
```sh
minikube start
kubectl config use-context minikube
```
2. Примените секрет (о нем читайте ниже в разделе `Kubernetes Secret`)

3. PostgreSQL внутри кластера
Установите PostgreSQL через Helm и задайте свои значения (логин/пароль/имя БД).
```sh
kubectl apply -f k8s/secret.yaml
```
После установки обновите секрет приложения с DATABASE_URL и перезапустите Django:
```sh
kubectl rollout restart deployment/django-deployment
```
5. Миграции
```sh
kubectl exec -it deploy/django-deployment -- python manage.py migrate --noinput
kubectl exec -it deploy/django-deployment -- python manage.py createsuperuser
```

## Kubernetes Secret

### 1) Как создать secret.yaml 
Проект ожидает чувствительные настройки (например `DATABASE_URL`, `SECRET_KEY`) через Kubernetes Secret.

1) Создайте локально файл k8s/secret.yaml, в нем будут лежать чувствительные данные.
Пример k8s/secret.yaml:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgres://USER:PASSWORD@HOST:5432/DBNAME" #Подставьте свои значения
  SECRET_KEY: "change_me" #Подставьте свои значения
  ALLOWED_HOSTS: "127.0.0.1,localhost" #Подставьте свои значения
```
2) Примените секрет:
```sh
kubectl apply -f k8s/secret.yaml
```

### Ingress (доступ к сайту по домену)

Ingress нужен, чтобы открывать сайт по доменному имени и стандартному порту 80, без ip в адресе.  
В этом проекте используется Ingress NGINX в Minikube.

1) Включите Ingress в Minikube
```bash
minikube addons enable ingress
```
2) Примените манифест Ingress
```bash
kubectl apply -f k8s/django-ingress.yaml
```
4) Пропишите домен в /etc/hosts/
Ingress будет доступен по домену star-burger.test

1. Узнать IP Minikube:
```bash
minikube ip
```
3. Откройте /etc/hosts/ и пропишите строку:
```
<MINIKUBE_IP> star-burger.test
```

После этого сайт должен открываться [http://star-burger.test](http://star-burger.test)

### Очистка устаревших сессий (clearsessions)
Это нужно, чтобы база не разрасталась из-за старых сессий и сайт работал стабильнее.
#### Запуск раз в месяц (CronJob)
1. Примените манифест:
```sh
kubectl apply -f k8s/clearsession-cronjob.yaml
```
2. Проверить, что создано:
```sh
kubectl get cronjob
```
#### Запустить прямо сейчас (вручную из CronJob)
1. Пропишите команду:
```sh
kubectl create job --from=cronjob/django-clearsessions django-clearsessions-once
```
2. Проверить, что создано:
```sh
kubectl get jobs
```
