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

# Запуск в Minikube

Сайт можно запустить на локальном компьютере с помощью связки [VirtualBox](https://www.virtualbox.org/) + [Minikube](https://minikube.sigs.k8s.io/docs/) + [K8s](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) - подробно инструкция описана в [видео](https://www.youtube.com/watch?v=WAIrMmCQ3hE&list=PLg5SS_4L6LYvN1RqaVesof8KAf-02fJSi&index=3).

Запустите Minikube командой:
```
minikube start
```

Создайте конфигурационный файл `django-app-config.yaml` в каталоге `kubernetes`, содержащий в себе все необходимые переменные окружения:
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
 kubectl apply -f kubernetes/django-app-config.yaml
```
Запустите сборку деплоймента:
```
kubectl apply -f kubernetes/django-app.yaml
```
Добавьте путь к тестовому хосту в файл `etc/hosts`:
```
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```
Включите сервисы `Ingress` в Minikube:
```
minikube addons enable ingress     
```
Запустите сборку `Ingress`:
```
kubectl apply -f kubernetes/ingress.yaml
```

Запустите манифест `clearsession`, для очистки сессий Django по расписанию:
```
kubectl apply -f kubernetes/clearsession.yaml
```
### Подключение PostgreSQL

Установите [Helm](https://helm.sh/docs/intro/install/)

Установите менеджер чартов Helm:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Создайте файл `postgres.yaml` с конфигурацией базы данных:
```
global:
  postgresql:
    auth:
      username: <USERNAME>
      password: <PASSWORD>
      database: <DB_NAME>
    service:
      ports:
        postgresql: 5432
```
Установите релиз PostgreSQL:
```
helm install postgresql -f kubernetes/postgresql.yaml bitnami/postgresql
```
Введите команду:
```
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.password}" | base64 -d)
```

Чтобы применить миграции к базе данных, используйте следующую команду:
```
kubectl apply -f kubernetes/django-migrate.yaml
```
Укажите `HOST` базы данных в `DATABASE_URL` манифеста `django-app-config.yaml`.


## Работа с сайтом

Откройте сайт по ссылке: http://star-burger.test/

# Запуск на внешнем кластере:

Создайте конфигурационный файл `django-app-config.yaml` в каталоге `kubernetes_prod`, содержащий в себе все необходимые переменные окружения:
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
 kubectl apply -f kubernetes_prod/django-app-config.yaml
```
Запустите сборку деплоймента:
```
kubectl apply -f kubernetes_prod/django-app.yaml
```

Установите сервисы `Ingress-контроллера` в кластере, например с помощью `nginx`:
```
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update

helm install nginx-ingress-tcp nginx-stable/nginx-ingress --set-string 'controller.config.entries.use-proxy-protocol=true' --create-namespace --namespace example-nginx-ingress-tcp
```

С помошью Helm установите [cert-manager](https://cert-manager.io/docs/installation/helm/).

В каталоге `kubernetes_prod` создайте манифест `cert.yaml`:

```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```
Примините манифест `cert.yaml`:
```
kubectl apply -f kubernetes_prod/cert.yaml
```
Запустите сборку `Ingress`(перед этим замените `HOST-name` на свои):
```
kubectl apply -f kubernetes_prod/ingress.yaml
```

Запустите манифест `clearsession`, для очистки сессий Django по расписанию:
```
kubectl apply -f kubernetes_prod/clearsession.yaml
```
### Подключение PostgreSQL

См. выше

## Работа с сайтом

Откройте сайт по ссылке: http://DOMAIN.EXAMPLE/

Где `DOMAIN.EXAMPLE` указанное в `ingress.yaml` доменное имя.

