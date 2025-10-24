# Инфраструктура в контейнерах
1. [Docker](#docker)
	1. [Установка Docker](#установка-docker)
	2. [Запуск контейнеров](#запуск-контейнеров)
	3. [Сборка образа контейнера](#сборка-образа-контейнера)
2. [Docker Compose](#docker-compose)
	1. [Работа с контейнерами](#работа-с-контейнерами)
	2. [Переменные окружения](#переменные-окружения)
3. [Автоматизация развёртки приложения](#автоматизация-развёртки-приложения)
	1. [Предварительные настройки окружения](#предварительные-настройки-окружения)
	2. [Bash-скрипт](#bash-скрипт)

---

##### Цель работы:
>Получение навыков работы с контейнерами при помощи Docker и Docker Compose, а также автоматизации развёртки приложений.

---

## Docker
### Установка Docker

>[!WARNING]
>Данная лабораторная работа выполняется на новой виртуальной машине, созданной по [аналогии с первой](Lab_1.md/#создание-новой-вм). Для удобства восприятия лабораторных имя пользователя второй виртуальной машины в методе изменено. У вас имя пользователя *должно быть в том же формате*, что и на первой ВМ.
>
>![](../images/lab_4/4.png)

>[!NOTE]
>В последней лабораторной работе рассмотрим другой подход развёртывания приложений и сопутствующей инфраструктуры, а именно — использование для этих задач изолированных сред: контейнеров. Мы будем использовать **Docker**  — это платформа для запуска приложений в изолированных средах (контейнерах).
>
>Docker-контейнеры — это самодостаточные программные модули, инкапсулирующие ПО и его зависимости: код, библиотеки, настройки окружения. В отличие от виртуальных машин, контейнеры используют ядро хост-системы и изолируют процессы на уровне ОС, что делает их более быстрыми и ресурсоэффективными. Это позволяет разработчикам создавать и тестировать приложения в одинаковых условиях независимо от окружения — будь то локальный компьютер, сервер или облачное хранилище.

Для начала установим Docker. Актуальное руководство по установке можно найти [здесь](https://docs.docker.com/engine/install/centos/). Рекомендуемый метод установки - настроить репозиторий и установить Docker из него.

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

```bash
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

![](../images/lab_4/4.1.png)

![](../images/lab_4/4.2.png)

После установки будет создана новая пользовательская группа с названием `docker`. Запустим службу Docker (заодно включив автозапуск) и добавим в неё своего пользователя:

```bash
sudo systemctl enable --now docker
```

```bash
sudo usermod -aG docker $USER
```

```bash
newgrp docker
```

![](../images/lab_4/4.3.png)

Перезапустим виртуальную машину и снова зайдём в группу `docker`.

```bash
newgrp docker
```

Проверить работоспособность Docker-а и правильно ли всё установлено можно командой:

```bash
docker container run hello-world
```

![](../images/lab_4/4.4.png)

>[!NOTE]
>Эту команду используют в первую очередь для проверки корректности установки и настройки Docker: она скачивает (если ещё не загружен) специальный тестовый образ `hello-world` с [Docker Hub](https://hub.docker.com/), запускает на его основе контейнер, который выводит простое приветственное сообщение в терминал, подтверждая, что Docker работает должным образом, а затем завершает свою работу.

Список доступных команд для работы с контейнерами в Docker можно посмотреть с помощью флага `--help`

```bash
docker container --help
```

![](../images/lab_4/4.5.png)

### Запуск контейнеров
>[!NOTE]
>В контейнерах запускаются процессы, определенные образами. Эти образы состоят из одного или нескольких слоев (или наборов различий) плюс некоторых метаданных. Один из способов взглянуть на образы и контейнеры — это рассматривать их как программы и процессы. Точно так же как процесс может рассматриваться «выполняемым приложением», контейнер может рассматриваться как образ, выполняемый докером.

 Запустим Docker-контейнер с уже знакомым нам веб-сервером NGINX, для этого выполним команду:

```bash
docker run -d nginx
```

![](../images/lab_4/4.6.png)

>[!NOTE]
>`-d` (от detached) — флаг, указывающий Docker запустить контейнер в фоновом режиме, то есть без привязки к текущему терминалу. Это позволяет продолжать работу в командной строке, пока контейнер работает в фоне. 
>
>`nginx` — имя официального образа веб-сервера NGINX из Docker Hub.
>
>В результате команда запускает контейнер с NGINX, который начинает слушать HTTP-запросы (по умолчанию на порту 80 внутри контейнера), работая незаметно в фоне.

Посмотреть список запущенных контейнеров можно командой:

```bash
docker container ls
```

А все контейнеры, даже остановленные — командой:

```bash
docker container ls -a
```

![](../images/lab_4/4.7.png)

>[!NOTE]
>В выводе команды отображаются следующие столбцы:
>- `CONTAINER ID` — уникальный идентификатор контейнера;
>- `IMAGE` — образ, на основе которого был создан контейнер;
>- `COMMAND` — команда, которая была выполнена при запуске контейнера;
>- `CREATED` — время, прошедшее с момента создания контейнера;
>- `STATUS` — текущее состояние контейнера;
>- `PORTS` — порты, которые использует контейнер;
>- `NAMES` — имена контейнеров (в данном случае случайно сгенерированные), удобны для ссылки на контейнер вместо ID.

При помощи команды `docker inspect <container id>` можно узнать множество информации о контейнере (и не только). Например, узнать IP-адрес контейнера можно так:

```bash
docker inspect 1acfd1ab9357 | grep IPAddress
```

Или с помощью:

```bash
docker inspect 1acfd1ab9357 | jq ".[0].NetworkSettings.IPAddress"
```

![](../images/lab_4/4.8.png)

Зная IP-адрес контейнера, можно отправить http-запрос при помощи утилиты **curl**:

```bash
curl http://172.17.0.2
```

![](../images/lab_4/4.9.png)

В ответ получим текст странички со стандартным приветствием NGINX.

Однако при попытке открыть в браузере страницу по адресу нашей виртуальной машины мы не обнаружим ожидаемого приветствия.

![](../images/lab_4/4.10.png)

Все дело в том, что по умолчанию контейнер запускается изолированно. Используемые сетевые порты доступны внутри изолированной сети Docker, но не снаружи узла, на котором запущен контейнер. Узнаем, прослушивается ли порт 80 хоста:

```bash
ss -tpln | grep 80
```

![](../images/lab_4/4.11.png)

Запустить контейнер с пробросом 80 порта хоста на 80 порт контейнера можно опцией `-p 80:80`, таким образом задав соответствие порта виртуальной машины (указывается первым) порту контейнера (указывается вторым).

```bash
docker container run -d -p 80:80 nginx
```

![](../images/lab_4/4.12.png)

Как видно, Docker не скачивает повторно образ NGINX, а запускает процесс из уже имеющегося образа в локальном репозитории.

В списке контейнеров и выводе `docker inspect` можно заметить открытые порты:

```bash
docker inspect ec35a67df1d4 | jq ".[0].NetworkSettings.Ports"
```

```bash
docker inspect 1acfd1ab9357 | jq ".[0].NetworkSettings.Ports"
```

![](../images/lab_4/4.13.png)

Откроем страницу по IP-адресу виртуальной машины:

![](../images/lab_4/4.14.png)

При запуске NGINX в контейнере возможно изменить его конфигурацию. Для этого можно примонтировать в контейнер директории файловой системы хоста при помощи опции `-v` (от слова volume). Создадим в домашней директории пользователя директорию `KTI_lab_4`, в ней два каталога: `conf` для конфигурационных файлов NGINX и `html` с html-файлами.

```bash
mkdir -p KTI_lab_4/{conf,html}
```

И проверим созданную структуру:

```bash
cd KTI_lab_4/
```

```bash
tree
```

![](../images/lab_4/4.15.png)

В директории `conf` создадим файл `default.conf` следующего содержания:

```bash
vi conf/default.conf
```

```nginx
server {
    listen 80;
    server_name _;

location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    }
}
```

![](../images/lab_4/4.16.png)

А в `html` файл `index.html`.

```bash
vi html/index.html
```

```html
<!DOCTYPE html> 
<html lang="en"> 

    <head> 
        <meta charset="UTF-8"> 
        <title>Hello!</title> 
    </head> 

    <body> 
        <h1>Hello from container!</h1> 
        <p>This is a simple paragraph.</p>
        <p>joker</p>
    </body> 

</html>
```

![](../images/lab_4/4.17.png)

Раньше для того, чтобы изменения вступили в силу, мы перезапускали приложение. То есть, останавливали процесс и запускали заново. В случае с контейнерами будем делать то же самое: останавливать контейнер, удалять его и запускать новый. Для остановки пропишем команду:

```bash
docker container stop $(docker container ls -q)
```

![](../images/lab_4/4.18.png)

>[!NOTE]
>Разберём команду по частям:
>- `docker container ls -q` — выводит только ID всех запущенных контейнеров (флаг `-q` означает quiet — тихий режим, без заголовков и лишней информации);
>- `$(...)` — это подстановка результата команды. То есть сначала выполняется `docker container ls -q`, и полученные ID передаются в качестве аргументов команде `docker container stop`;
>- `docker container stop` — корректно завершает указанные контейнеры.

Удалить все контейнеры можно аналогично:

```bash
docker container rm $(docker container ls -aq)
```

Проверим, остались ли контейнеры.

```bash
docker container ls -a
```

![](../images/lab_4/4.19.png)

После того, как все контейнеры удалены, можно вернуться к запуску NGINX с пользовательским конфигом и страничкой приветствия. Команда для запуска контейнера NGINX с двумя примонтированными директориями примет вид:

```bash
docker container run -d -p 80:80 --rm --name nginx -v '/home/joker/KTI_lab_4/conf:/etc/nginx/conf.d' -v '/home/joker/KTI_lab_4/html:/usr/share/nginx/html' nginx
```

![](../images/lab_4/4.20.png)

>[!NOTE]
>Разберём новые аргументы команды запуска:
>- `--rm` — автоматически удаляет контейнер после его остановки (удобно для временных или тестовых запусков);
>- `--name nginx` — присваивает контейнеру имя `nginx` (вместо случайно сгенерированного);
>- `-v '/home/joker/KTI_lab_4/conf:/etc/nginx/conf.d'` — монтирует локальную директорию `/KTI_lab_4/conf` с конфигурационными файлами NGINX в контейнер по пути `/etc/nginx/conf.d`;
>- `-v '/home/joker/KTI_lab_4/html:/usr/share/nginx/html'` — монтирует локальную директорию `/KTI_lab_4/html` с html-файлами в контейнер по пути `/usr/share/nginx/html`.

Попробуем открыть сайт по IP-адресу виртуальной машины:

![](../images/lab_4/4.21.png)

Веб-сервер функционирует, однако команда для запуска контейнера становится все более и более громоздкой.

### Сборка образа контейнера
>[!NOTE]
>Можно пойти другим путем и собрать образ, включив в него файлы нашего приложения. Образ контейнера (**Docker image**) — это статичный шаблон, содержащий всё необходимое для запуска приложения: код, библиотеки, зависимости, системные инструменты, настройки и команду по умолчанию для запуска. Образ состоит из слоёв (layers) — каждая инструкция в Dockerfile создаёт новый слой, который содержит только изменения относительно предыдущего. Благодаря этому образы легко переиспользуются и быстро собираются.

В каталоге `KTI_lab_4` создадим файл с именем `Dockerfile` без расширения.

```bash
vi Dockerfile
```

Содержимое файла:

```docker
FROM nginx:1.29.2

COPY ./conf /etc/nginx/conf.d 
COPY ./html /usr/share/nginx/html
```

![](../images/lab_4/4.22.png)

>[!NOTE]
>Разберём каждую строку:
>- `FROM nginx:1.29.2` — использует официальный образ NGINX версии 1.29.2 в качестве базы для нового образа;
>- `COPY ./conf /etc/nginx/conf.d` — копирует локальную папку `./conf` в директорию `/etc/nginx/conf.d` внутри контейнера;
>- `COPY ./html /usr/share/nginx/html` — копирует локальную папку `./html` в директорию `/usr/share/nginx/html`.

Теперь соберём образ из файла находясь в каталоге `KTI_lab_4` (точка в конце соответствующей команды нужна).

```bash
docker buildx build -t joker/nginx:1.0 .
```

![](../images/lab_4/4.23.png)

Созданный образ появится в локальном репозитории Docker-образов.

```bash
docker image ls
```

![](../images/lab_4/4.24.png)

Остановим старый контейнер с NGINX.

```bash
docker container stop nginx
```

И из только что собранного образа запустим контейнер командой:

```bash
docker run -d -p 80:80 --rm --name nginx joker/nginx:1.0
```

![](../images/lab_4/4.25.png)

Можно проверить содержимое интересующих нас файлов в контейнере.

```bash
docker container exec nginx cat /etc/nginx/conf.d/default.conf
```

```bash
docker container exec nginx cat /usr/share/nginx/html/index.html
```

![](../images/lab_4/4.26.png)

Проверим работоспособность запущенного контейнера.

![](../images/lab_4/4.28.png)

И перед следующим этапом опять остановим контейнер с NGINX.

```bash
docker container stop nginx
```

![](../images/lab_4/4.29.png)

---

## Docker Compose
### Работа с контейнерами
>[!NOTE]
>**Docker Compose** — это инструмент для управления многоконтейнерными приложениями с помощью одного файла конфигурации в формате **YAML** (обычно `docker-compose.yml`), в котором описываются все сервисы — такие как веб-сервер, база данных и т.д. — вместе с их образами, портами, томами, переменными окружения и зависимостями. Вместо того чтобы запускать каждый контейнер отдельной командой, разработчик может развернуть всю систему одной командой `docker compose up` и остановить командой `docker compose down`.

Проверить версию и факт установки Docker Compose можно так:

```bash
docker compose version
```

![](../images/lab_4/4.30.png)

Файл с инструкциями `docker-compose.yaml` создадим в директории `KTI_lab_4`.

```bash
vi docker-compose.yaml
```

И заполним следующим образом:

```docker
name: lab4
services:
  nginx:
    image: "nginx:1.29.2"
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./conf:/etc/nginx/conf.d
      - ./html:/usr/share/nginx/html
```

![](../images/lab_4/4.31.png)

При написании файлов формата YAML следует внимательно относиться к отступам — они являются частью синтаксиса. Проверить конфигурацию на корректность можно командой:

```bash
docker compose config
```

![](../images/lab_4/4.32.png)

Теперь запустим контейнер:

```bash
docker compose up
```

![](../images/lab_4/4.33.png)

Загрузки веб-страницы будут логироваться и выводиться в консоль.

![](../images/lab_4/4.34.png)

Выйти из просмотра логов и остановить выполнение можно при помощи сочетания клавиш `Ctrl+C`. Запуск Docker Compose в фоновом режиме осуществляется добавлением в состав команды запуска ключа `-d`.

Для следующего этапа проекта соберем новый образ — на основе Python с фреймворком Flask. Для этого возьмем за основу официальный образ Python и соберем новый, добавив новым слоем Flask. Изменим имеющийся `Dockerfile`:

```bash
vi Dockerfile
```

```docker
FROM python:3.13-alpine

RUN pip install --no-cache-dir --upgrade pip flask
WORKDIR /app
```

![](../images/lab_4/4.35.png)

И соберем образ.

```bash
docker buildx build -t joker/flask:1.0 .
```

![](../images/lab_4/4.36.png)

Создадим в директории `KTI_lab_4` каталог `app`, а в ней файл `app.py`.

```bash
mkdir app && vi app/app.py
```

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return "<h1>Hello from Flask!<h1>"

if __name__ == '__main__':
   app.run(host='0.0.0.0')
   
```

![](../images/lab_4/4.37.png)

Изменим файл `default.conf`:

```bash
vi conf/default.conf
```

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://flask:5000;
    }
}
```

![](../images/lab_4/4.38.png)

Добавим в файл `docker-compose.yaml` сервис `flask`:

```bash
vi docker-compose.yaml
```

```docker
name: lab4
services:
  nginx:
    image: "nginx:1.29.2"
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./conf:/etc/nginx/conf.d
      - ./html:/usr/share/nginx/html

  flask:
    image: "joker/flask:1.0"
    container_name: flask
    environment:
      - FLASK_DEBUG=True
      - PYTHONUNBUFFERED=True
    volumes:
      - ./app:/app
    ports:
      - "5000:5000"
    command: python app.py
```

![](../images/lab_4/4.39.png)

На данном этапе структура проекта должна выглядеть так:

![](../images/lab_4/4.40.png)

![](../images/lab_4/4.41.png)

Запустим.

```bash
docker compose up -d 
```

![](../images/lab_4/4.42.png)

И посмотрим страничку доступную теперь по IP-адресу ВМ.

![](../images/lab_4/4.43.png)

Логи контейнеров можно посмотреть командой:

```bash
docker compose logs -f
```

![](../images/lab_4/4.44.png)

>[!NOTE]
>Флаг `-f` отвечает за отображение логов в реальном времени.

Теперь добавим в проект **PostgreSQL** в качестве СУБД. Для этого добавим еще один сервис в файл `docker-compose.yaml`.

```bash
vi docker-compose.yaml
```

```docker
name: lab4
services:
  nginx:
    image: "nginx:1.29.2"
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./conf:/etc/nginx/conf.d
      - ./html:/usr/share/nginx/html
    depends_on:
      - flask

  flask:
    image: "joker/flask:1.0"
    container_name: flask
    environment:
      - FLASK_DEBUG=True
      - PYTHONUNBUFFERED=True
    volumes:
      - ./app:/app
    ports:
      - "5000:5000"
    command: python app.py
    depends_on:
      - postgres

  postgres:
    image: "postgres:18-alpine"
    container_name: postgres
    environment:
      - POSTGRES_PASSWORD=Password!
      - POSTGRES_USER=flask_user
      - POSTGRES_DB=flask_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

![](../images/lab_4/4.45.png)

После запуска увидим запущенный контейнер с PostgreSQL.

```bash
docker compose up -d
```

![](../images/lab_4/4.46.png)

Для диагностики бывает полезно подключиться к командной оболочке запущенного контейнера в интерактивном режиме. Это можно сделать разными способами. Один из них — подключение из командной строки. Попробуем подключиться к сервису `postgres`:

```bash
docker compose exec -u postgres postgres bash
```

Внутри зайдем в консоль БД.

```bash
psql -U flask_user -d flask_db
```

И введём команду, которая показывает все развернутые базы данных.

```postgresql
 \l
```

![](../images/lab_4/4.47.png)

Еще один вариант подключиться к контейнеру — воспользоваться расширением Docker для VS Code:

![](../images/lab_4/4.48.png)

После установки данного расширения и перехода в соответствующую вкладку увидим следующее:

![](../images/lab_4/4.49.png)

Нажмём правой кнопкой мыши на интересующий нас контейнер и в выпадающем меню выберем пункт `Attach Shell`.

![](../images/lab_4/4.50.png)

После чего автоматически откроется консоль нашего контейнера. Где мы снова можем выполнить те же команды что и в предыдущем варианте:

```bash
psql -U flask_user -d flask_db
```

```postgresql
 \l
```

![](../images/lab_4/4.51.png)

>[!NOTE]
>Существует множество способов и ПО для подключения как к контейнерам так и к базам данных, здесь перечислена лишь малая часть.
>
>Удобными инструментами для администрирования СУБД являются [DBeaver](https://dbeaver.io/download/), [pgAdmin](https://www.pgadmin.org/download/) и т.п.

### Переменные окружения
>[!NOTE]
>Переменные окружения — это способ передачи конфигурационных данных контейнерам без изменения кода приложения. Они задаются в файле `docker-compose.yml` с помощью ключа `environment` (что вы могли заметить в предыдущих этапах данной лабораторной работы), либо загружаются из внешнего файла (обычно с расширением `.env`) с помощью директивы `env_file`. Переменные окружения позволяют гибко управлять настройками приложения — такими как адрес базы данных, порты, режим отладки и другие параметры. Это делает приложение более гибким и безопасным, поскольку чувствительные данные не хранятся напрямую в коде.
>
>Файлы секретов — это механизм безопасного хранения и передачи конфиденциальной информации, такой как пароли, токены API или ключи шифрования. В отличие от переменных окружения, которые могут быть видны в процессах или логах контейнера, секреты монтируются в контейнеры как файлы в защищённой директории. В `docker-compose.yml` секреты объявляются в секции `secrets`. Использование секретов снижает риск утечки чувствительных данных.

Создадим в `KTI_lab_4` 3 файла, в которые вынесем переменные окружения и пароль:

```bash
vi flask.env
```

```conf
FLASK_DEBUG=True
PYTHONUNBUFFERED=True
```

![](../images/lab_4/4.52.png)

```bash
vi postgres.env
```

```conf
POSTGRES_PASSWORD_FILE=/run/secrets/db_password
POSTGRES_USER=flask_user
POSTGRES_DB=flask_db
```

![](../images/lab_4/4.53.png)

```bash
vi db_password
```

```conf
Password!
```

![](../images/lab_4/4.54.png)

После чего внесём изменения в конфигурационный файл Docker Compose.

```bash
vi docker-compose.yaml
```

```docker
name: lab4
services:
  nginx:
    image: "nginx:1.29.2"
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./conf:/etc/nginx/conf.d
      - ./html:/usr/share/nginx/html
    depends_on:
      - flask

  flask:
    image: "joker/flask:1.0"
    container_name: flask
    env_file:
      - flask.env
    volumes:
      - ./app:/app
    ports:
      - "5000:5000"
    command: python app.py
    depends_on:
      - postgres

  postgres:
    image: "postgres:18-alpine"
    container_name: postgres
    env_file:
      - postgres.env
    secrets:
      - db_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

secrets:
  db_password:
    file: ./db_password

volumes:
  postgres_data:
```

![](../images/lab_4/4.55.png)

Чтобы пересоздать контейнеры, воспользуемся командой:

```bash
docker compose down && docker compose up -d
```

![](../images/lab_4/4.56.png)

Проверить, что пароль из файла секретов действительно подходит к базе можно, попытавшись подключиться к ней.

Перед началом следующей части лабораторной работы удалим созданные контейнеры и тома.

```bash
docker compose down
```

```bash
docker volume rm lab4_postgres_data
```

![](../images/lab_4/4.57.png)

---

## Автоматизация развёртки приложения
### Предварительные настройки окружения
>[!NOTE]
>В этой части лабораторной мы научимся автоматизировать развёртку приложений на примере уже знакомого нам сайта регистрации.

Сперва подготовим инфраструктуру для него. В домашнем каталоге создадим папку `KTI_infra`, а в ней `Dockerfile`:

```bash
mkdir KTI_infra && vi KTI_infra/Dockerfile
```

```docker
FROM python:3.12-alpine

WORKDIR /flask_project

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

RUN pip install --upgrade pip
COPY ./flask_project/requirements.txt /flask_project/requirements.txt
RUN pip install -r requirements.txt

COPY ./flask_project /flask_project
```

```bash
cd KTI_infra
```

![](../images/lab_4/4.58.png)

Так же создадим файлы с переменными окружения.

```bash
vi flask.env
```

```conf
SQL_DATABASE_TYPE=postgres
SQL_USER=flask_user
SQL_PASSWORD_FILE=/run/secrets/db_password
SQL_HOST=postgres
SQL_PORT=5432
SQL_DATABASE=flask_db
```

![](../images/lab_4/4.59.png)

```bash
vi postgres.env
```

```conf
POSTGRES_PASSWORD_FILE=/run/secrets/db_password
POSTGRES_USER=flask_user
POSTGRES_DB=flask_db
```

![](../images/lab_4/4.60.png)

Ещё для работы приложения понадобится создать папку `conf` с файлом конфигурации NGINX:

```bash
mkdir conf && vi conf/default.conf
```

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://flask:5000;
    }
}
```

![](../images/lab_4/4.61.png)

Теперь создадим "сердце" проекта:

```bash
vi docker-compose.yaml
```

```docker
name: lab4
services:
  nginx:
    image: "nginx:1.29.2"
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./conf:/etc/nginx/conf.d
    depends_on:
      - flask

  flask:
    # image: "joker/flask:1.0"
    build: .
    container_name: flask
    env_file:
      - flask.env
    ports:
      - "5000:5000"
    secrets:
      - db_password
    command: flask --app flask_app run --host=0.0.0.0
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: "postgres:18-alpine"
    container_name: postgres
    env_file:
      - postgres.env
    ports:
      - "5432:5432"
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d flask_db -U flask_user"]
      interval: 10s
      timeout: 5s
      retries: 5

secrets:
  db_password:
    file: ./db_password

volumes:
  postgres_data:
```

![](../images/lab_4/4.62.png)

В конце создадим файл `.gitignore` следующего содержания:

```bash
vi .gitignore
```

```conf
/flask_project
db_password
```

![](../images/lab_4/4.63.png)

Для того, чтобы иметь возможность разворачивать наш проект в любом месте мы поместим все необходимые файлы на GitHub. В целях разграничения кода инфраструктуры и кода приложения будут созданы два репозитория. Первым создадим репозиторий инфраструктуры c названием `KTI_infra` (подробно процесс создания и настройки удалённого репозитория был описан в [предыдущей лабораторной](Lab_3.md/#использование-github)) и выгрузим в него файлы описания инфраструктуры.

![](../images/lab_4/4.64.png)

В итоге будем иметь первый репозиторий такого вида:

![](../images/lab_4/4.65.png)

Ещё нужно создать файл со случайно сгенерированным паролем, это можно сделать командой:

```bash
tr -dc A-Za-z0-9 </dev/urandom | head -c 24 > db_password
```

После этого скачаем в `KTI_infra` сам код приложения и перейдём в директорию `flask_project`:

```bash
git clone https://github.com/kottik-mypp/flask_project.git
```

```bash
cd flask_project
```

![](../images/lab_4/4.66.png)

После чего повторим действия по загрузке в удалённый репозиторий с названием `flask_project` теперь уже кода приложения.

![](../images/lab_4/4.67.png)

Как будет выглядеть второй репозиторий на GitHub:

![](../images/lab_4/4.68.png)

Теперь мы можем запустить наше приложение.

```bash
docker compose up -d
```

![](../images/lab_4/4.69.png)

Нам остался последний шаг. Настроим первую виртуальную машину так, что она будет передавать запросы сделанные на её IP-адрес на вторую ВМ. В качестве основного балансировщика нагрузки будет выступать NGINX, настроенный в предыдущей работе. С него запросы будут направляться на виртуальную машину с развернутыми контейнерами. 

Для настройки ***вернемся на первую ВМ*** и изменим конфигурационный файл NGINX:

```bash
cd /etc/nginx/
```

```bash
mv ./conf.d/flask_app.conf ./conf.d/flask_app.conf_old
```

```bash
vi ./conf.d/balancer.conf
```

```nginx
upstream backend { 
    server 192.168.243.130:80;
}

server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;
    include ssl_params;

    location / {
        proxy_pass http://backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr; 
    }
}
```

![](../images/lab_4/4.70.png)

>[!NOTE]
>В директиве `server` указываем IP-адрес второй виртуальной машины.

После чего перезагружаем NGINX.

```bash
sudo nginx -s reload
```

И теперь откроем страничку в браузере по IP-адресу первой ВМ:

![](../images/lab_4/4.71.png)

Всё работает, а если посмотреть логи Docker-а, то можно увидеть запросы с первой ВМ.

### Bash-скрипт
>[!NOTE]
>Bash-скрипты — это текстовые файлы, содержащие последовательность команд для оболочки Bash, стандартной командной оболочки в большинстве Linux-систем. Скрипты позволяют автоматизировать рутинные задачи, такие как управление файлами, установка программ, обработка данных или настройка системы, объединяя множество команд в один исполняемый файл.

В домашней директории создадим файл с нашим скриптом автоматизации:

```bash
vi create-infra.sh
```

И вставим в него:

```bash
#!/usr/bin/env bash

REPO_USER=Porfik
GREEN='\033[92m'
RED='\033[91m'
NC='\033[0m'

function date_f {
    date "+%d.%m.%Y %H:%M:%S"
}

function generate_secret {
    secret=$(tr -dc A-Za-z0-9 </dev/urandom | head -c 24; echo)
    echo "$secret" > db_password
    red_text "Database password: $secret"
}

function green_text() {
    printf "${GREEN}%s${NC}\n" "$1$2"
}

function red_text() {
    printf "${RED}%s${NC}\n" "$1$2"
}

function rainbow_text() {
    TEXT="$1"
    COLORS=('\033[91m' '\033[92m' '\033[93m' '\033[94m' '\033[95m' '\033[96m')
    for ((i=0; i<${#TEXT}; i++)); do
        printf "${COLORS[i % ${#COLORS[@]}]}%s${NC}" "${TEXT:i:1}"
    done
    printf "\n"
}

green_text "$(date_f) " "Cleanup started"

existing_containers=$(docker container ls -aq)
if [ -n "$existing_containers" ]; then
    running_containers=$(docker container ls -q)
    if [ -n "$running_containers" ]; then
        docker container stop $running_containers
    fi
    docker container rm $existing_containers
fi

docker buildx prune -af

all_images=$(docker image ls -aq)
if [ -n "$all_images" ]; then
    docker image rm -f $all_images
fi

all_volumes=$(docker volume ls -q)
if [ -n "$all_volumes" ]; then
    docker volume rm $all_volumes
fi

rm -rf ~/KTI_infra
green_text "$(date_f) " "Cleanup finished"
green_text "$(date_f) " "Cloning infrastructure from repo"

git clone https://github.com/$REPO_USER/KTI_infra.git
cd KTI_infra

green_text "$(date_f) " "Cloning project files from repo"
git clone https://github.com/$REPO_USER/flask_project.git
green_text "$(date_f) " "Starting containers"

generate_secret
docker compose up -d

rainbow_text "$(date_f) Infrastructure ready"

exit 0
```

![](../images/lab_4/4.72.png)

В скрипте необходимо исправить значение переменной окружения `REPO_USER`, подставив в нее имя вашей учётной записи на GitHub.

После создания необходимо добавить файлу со скриптом права на исполнение.

```bash
chmod +x create-infra.sh
```

![](../images/lab_4/4.73.png)

И теперь можно его запускать:

```bash
./create-infra.sh
```

![](../images/lab_4/4.74.png)

![](../images/lab_4/4.75.png)

![](../images/lab_4/4.76.png)

Зарегистрируемся и войдём на сайт:

![](../images/lab_4/4.77.png)

Скрипт работает, и сайт запускается, а это значит, что вы закончили 4 лабораторную работу!!!

Теперь вам осталось пройти через самую увлекательную часть учебного процесса — ЗАЩИТУ ЛАБАРАТОРНЫХ. Удачи!