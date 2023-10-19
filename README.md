# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).


## Как задеплоить проект в Kubernetes (Minikube)

Переходим в каталог проекта  

```shell-session
cd <диск:папка с проектами>\test-django\backend_main_django
```

Создайте образ проекта  

```shell-session
minikube delete                         # подготовка Minikube
minikube start                          # ...
minikube image build -t django-app:1 .
minikube image ls --format=table        # посмотреть результат
```

Создайте в папке `kubernetes` файл `configmap.yaml` со следующим содержимым:   
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config-file
data:
  SECRET_KEY: "секретный ключ Django"
  DEBUG: "False" 
  ALLOWED_HOSTS: "ваш IP"
  DATABASE_URL: "postgres://postgres:PASSWORD@DOMEN.RU:PORT/DB_NAME"    # данные БД
  DJANGO_SUPERUSER_PASSWORD: "ваш пароль"
  DJANGO_SUPERUSER_EMAIL: "ваш email"
  DJANGO_SUPERUSER_USERNAME: "ваше имя админа"
```

Подключите внешний сервис minikube ingress для доступа по внешнему IP
```shell-session
minikube addons enable ingress
```

Установте приложение [HELM](https://helm.sh/) 
В каталоге проекта запустите:
```shell-session
helm install <версия деплоя> ../chart/
```

или без установки HELM - поочередно следующие команды:   
```shell-session
kubectl apply -f ../kubernetes/ingress-hosts.yaml             # добавляем ingress rules для маршрутизации обращений 
kubectl apply -f ../kubernetes/configmap.yaml                 # создаём/обновляем переменные ConfigMap
kubectl apply -f ../kubernetes/test-django-deployment.yaml    # создаем деплоймент проекта с подами, сервисом
kubectl apply -f ../kubernetes/app-migrations.yaml            # подготавливаем БД (если подключаемся к пустой БД)
kubectl apply -f ../kubernetes/django-clearsessions.yaml      # создаем задачу ежедневного удаления старых сессий Django    
```

Получите IP-адрес сайта и занесите его в [файл hosts на вашем сервере: ](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit)
```shell-session
kubectl get ingress
```
В hosts с правами администратора добавляем в конце запись вида  
```
192.168.59.107 star-burger.test
```

Если обновились какие либо переменные в `configmap.yaml`:  

```shell-session
kubectl apply -f ../kubernetes/configmap.yaml                 # создаём/обновляем переменные ConfigMap
kubectl get pods                                              # получаем имена подов
kubectl delete pod <имя пода>                                 # удаляем запущенный под с автоматическим стартом нового с новыми переменными  
```


## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Цели проекта

Код написан в учебных целях — это урок в курсе по Python и веб-разработке на сайте [Devman](https://dvmn.org).
