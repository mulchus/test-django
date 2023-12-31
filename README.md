# Django site

Докеризированный [сайт](https://edu-stoic-dubinsky.sirius-k8s.dvmn.org/) на Django для экспериментов с Kubernetes.

## Как запустить dev-версию в Kubernetes (Minikube)

Переходим в каталог проекта  

```shell-session
cd <диск:папка с проектами>\test-django\backend_main_django
```

Создайте образ проекта или используйте имеющийся образ из DockerHub.  

Создание образа проекта:  
```shell-session
minikube delete                         # подготовка Minikube
minikube start                          # ...
minikube image build -t django-app:1 .
minikube image ls --format=table        # посмотреть результат
```

Загрузка образа проекта из DockerHub:  
```shell-session
docker login                                # подключиться к DockerHub
docker images                               # посмотреть имеющиеся локально образы
docker pull <mulchus/django-app:cec55e78>   # загрузить из DockerHub образ (<пример имени:тэга>)
docker images                               # посмотреть результат
```

Создайте в папке `kubernetes` файл `configmap.yaml` со следующим содержимым:   
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config-file-{{ .Release.Name }}
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
и проверьте результат установки:
```shell-session
kubectl get all -n ingress-nginx
```

Для деплоя одной командой через HELM - установте приложение [HELM](https://helm.sh/)   
Уточните название вышесформированного образа проекта в файле [values.yaml](chart%2Fvalues.yaml):
```shell-session
...
container:
  image: mulchus/django-app:cec55e78        # пример имени
...
```
Скопирцуйте из папки `kubernetes` в папку `chart/templates` файл `configmap.yaml`
В каталоге проекта запустите:
```shell-session
helm install <произвольная версия деплоя> ../chart/
```

Или деплой без установки HELM:  
Уточните название вышесформированного образа проекта в файле [kubernetes/test-django-deployment.yaml](kubernetes%2Ftest-django-deployment.yaml):
```shell-session
...
      containers:
        - name: django-app
          image: mulchus/django-app:cec55e78        # пример имени
...
```
- и введите поочередно следующие команды:   
```shell-session
kubectl apply -f ../kubernetes/ingress-hosts.yaml             # добавляем ingress rules для маршрутизации обращений 
kubectl apply -f ../kubernetes/configmap.yaml                 # создаём/обновляем переменные ConfigMap
kubectl apply -f ../kubernetes/test-django-deployment.yaml    # создаем деплоймент проекта с подами, сервисом
kubectl apply -f ../kubernetes/app-migrations.yaml            # подготавливаем БД (если подключаемся к пустой БД)
kubectl apply -f ../kubernetes/django-clearsessions.yaml      # создаем задачу ежедневного удаления старых сессий Django    
```

Проверьте результат запуска:  
```shell-session
kubectl get all
```
и при наличии ошибок - подробно изучите детализацию конкретного элемента системы:  
например pod'a `test-django-pod-v1-6bd95b7f45-2s4pc`
```shell-session
kubectl describe pod/test-django-pod-v1-6bd95b7f45-2s4pc
```
    

Получите IP-адрес сайта и занесите его в [файл hosts на вашем сервере: ](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit)
```shell-session
kubectl get ingress
```
В hosts с правами администратора добавляем в конце запись вида  
```
192.168.59.107 star-burger.test
```

Если обновились какие либо переменные в `configmap.yaml`, выполняем:  
```shell-session
kubectl apply -f ../kubernetes/configmap.yaml                 # создаём/обновляем переменные ConfigMap
kubectl get pods                                              # получаем имена подов
kubectl delete pod <имя пода>                                 # удаляем запущенный под с автоматическим стартом нового с новыми переменными  
```

При необходимости сменить образ в работающем сайте, выполняем:  
```shell-session
kubectl set image deployment/test-django-pod-v1 django-app-v1=mulchus/django-app:<хэш-тэг новой версии>
```


## Как запустить prod-версию на узле от Devman'a в Yandex.Cloud 

[Описание выделенных ресурсов облачной инфраструктуры](https://sirius-env-registry.website.yandexcloud.net/edu-stoic-dubinsky.html)

Выполните вышеуказанные действия для деплоя dev-версии, за исключением следующего:

1. Не создаем образ проекта, а используем готовый образ с тэгом '14cc0bfd' (mulchus/django-app:14cc0bfd)
2. Пропускаем раздел `Подключите внешний сервис minikube ingress для доступа по внешнему IP`
3. Выполняем раздел `Для деплоя одной командой через HELM...`
4. Проверячем результат деплоя на нашем [домене](https://edu-stoic-dubinsky.sirius-k8s.dvmn.org/) 



## Как запустить dev-версию в контейнере на сервере VPN под Linux

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

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
