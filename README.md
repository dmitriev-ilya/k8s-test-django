# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск в Minikube

Сайт можно запустить на локальном компьютере с помощью связки [VirtualBox](https://www.virtualbox.org/) + [Minikube](https://minikube.sigs.k8s.io/docs/) + [K8s](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) - подробно инструкция описана в [видео](https://www.youtube.com/watch?v=WAIrMmCQ3hE&list=PLg5SS_4L6LYvN1RqaVesof8KAf-02fJSi&index=3).

Запустите Minikube командой:
```
minikube start
```
Введите команду ниже, чтобы получить возможность использовать Docker внутри кластера:
```
eval $(minikube docker-env)
```
Затем соберите Docker-образ нашего Django-приложения внутри кластера:
```
docker build -t django-app ./backend_main_django
```
Создайте конфигурационный файл `django-app-config.yaml`, содержащий в себе все необходимые переменные окружения:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
  labels:
    app: django-app
data:
  SECRET_KEY: <django secret key>
  DATABASE_URL: postgres://USERNAME:PASSWORD@HOST:PORT/DB_NAME
  DEBUG=False
  ALLOWED_HOSTS=<список доступных хостов>
```
Запустите файл-конфиг:
```
 kubectl apply -f django-app-config.yaml
```
Запустите сборку деплоймента:
```
kubectl apply -f django-app.yaml
```
Добавьте путь к тестову хосту в файл `etc/hosts`:
```
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```
Включите сервисы `Ingress` в Minikube:
```
minikube addons enable ingress     
```
Запустите сборку `Ingress`:
```
kubectl apply -f ingress.yaml
```

Запустите манифест `clearsession`, для установки очистки сессий Django по расписанию:
```
kubectl apply -f clearsession.yaml
```

Откройте сайт по ссылке: http://star-burger.test/
