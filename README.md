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

## Размещение проекта в локальном кластере Kubernetes
Для размещения проекта понадобится локальный кластер [`minikube`](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download). А также установленные инструменты: [`kubectl`](https://kubernetes.io/docs/tasks/tools/) и [`helm`](https://helm.sh/).  
Создайте образ проекта и загрузите его в кластер.
```sh
$ docker compose build
$ minikube image load django_app:latest
```
Создайте под (pod) postgresql с указанием базы данных логина и пароля, которые будут использоваться проектом. В выводе комманды будет указан адрес сервера `postgre` - `djapp-db-postgresql.default.svc.cluster.local`
```sh
$ helm install djapp-db oci://registry-1.docker.io/bitnamicharts/postgresql --set
global.postgresql.auth.username=user --set
global.postgresql.auth.password=pass --set
global.postgresql.auth.database=base
```
Перед запуском проекта необходимо в кластере создать объект `secret` с чувствительными данными.  
Отредактируйте файл `kubernetes\djapp-secret.yml` и выполните команду
```sh
$ kubectl apply -f djapp-secret.yml
```
Затем запустить проект используя манифест файл из папки `kubernetes`
```sh
$ kubectl apply -f djapp-v1.yml
```
Для организации внешнего доступа к проекту по доменному имени используется контроллер Ingress.  
Добавьте необходимые доменные имена в файле `djapp-v1.yml`
```sh
...
- name: ALLOWED_HOSTS
  value: "127.0.0.1,localhost,192.168.49.2,star-burger.test"
...
```
и файле правил `djapp-ingress.yml`
```sh
...
rules:
- host: star-burger.test
...
```
и запустите правила в работу
```sh
$ kubectl apply -f djapp-ingress.yml
```
Для первоначального заполнения и при обновлении базы данных, миграцию выполнить коммандой
```sh
$ kubectl apply -f djapp-migrate.yml
```
Для создания суперпользователя подключаемся к любому из запущенных подов
```sh
$ kubectl exec pod/djapp-deploy-[id] -it -- bash
root@pod$ python manage.py createsuperuser
```
Добавление регулярной задачи очистки сессий раз месяц
```sh
$ kubectl apply -f djapp-clearsessions-cron.yml
```
Можно вручную запустить очистку сессий вне расписания
```sh
$ kubectl create job --from=cronjob/djapp-clearsessions-cron djapp-clearsessions-onetime
```
