# Выпуск TLS-сертификатов и работа с Git-репозиториями
1. [Настройка TLS](#настройка-tls)
	1. [Создание сертификатов](#создание-сертификатов)
	2. [Настройка веб-сервера и хоста](#настройка-веб-сервера-и-хоста)
2. [Работа с Git](#работа-с-git)
	1. [Основы Git](#основы-git)
	2. [Принципы использования веток](#принципы-использования-веток)
	3. [Использование GitHub](#использование-github)

---

##### Цель работы:
> Получить навыки по выпуску сертификатов, работе с Git-репозиториями.

---

## Настройка TLS
### Создание сертификатов
>[!NOTE]
>Прежде чем браться за настройку TLS, следует разобраться с таким понятием как **PKI**. PKI (Public Key Infrastructure) – инфраструктура открытых ключей, имеет множество вариантов применения, но в основном это шифрование и подпись данных. PKI основана на асимметричном шифровании (это когда для шифрования данных используется один ключ, а для расшифрования - другой). В большинстве случаев данные шифруются закрытым ключом, который хранится в секрете и не распространяется (private key), а расшифровываются на принимающей стороне заранее полученным открытым ключом (public key).
>
>Открытый ключ распространяется не сам по себе, а в составе сертификата (сертификата x.509). Это что-то вроде паспорта, он содержит информацию, позволяющую идентифицировать субъект, которому выдан сертификат (поле `Subject`), указывает, кем он был выпущен (поле `Issuer`), серийный номер сертификата и многое другое. Сертификат можно сгенерировать и подписать самостоятельно (такой сертификат будет называться самоподписанным), а можно приобрести в удостоверяющем центре (УЦ) – организации, специализирующейся на изготовлении сертификатов и электронных цифровых подписей. В первом случае преимуществом будет высокая скорость изготовления, низкие затраты, во втором – сертификат будет приниматься не только на нашем оборудовании, но и у других пользователей.

В данной работе поступим следующим образом – создадим свой локальный УЦ, выпустим его корневой сертификат, который добавим в хранилище доверенных корневых сертификатов на хосте. Это позволит принимать дочерние сертификаты, подписанные нашим УЦ. После чего создадим запрос на сертификат веб-сервера и подпишем его закрытым ключом нашего УЦ. Таким образом получим сертификат с открытым ключом веб-сервера, который будет использовать NGINX при HTTPS-соединении.

Для наглядности и во избежание ошибок каждый файл (ключ, запрос на сертификат и сам сертификат) будем генерировать отдельной командой, используя конфигурационные файлы по мере необходимости. Начнем с закрытого ключа удостоверяющего центра. Но в начале организуем место хранения наших ещё не созданных сертификатов.

```bash
mkdir -p ./pki/{certs,newcerts,private}
```

```bash
cd pki
```

Созданную структуру можно посмотреть используя команду:

```bash
tree
```

![](../images/lab_3/3.0.png)

Теперь можем сгенерировать сам ключ.

```bash
openssl genrsa -out ./private/ca.key 4096
```

![](../images/lab_3/3.1.png)

Мы получили файл с именем `ca.key`, в котором содержится закрытый ключ нашего УЦ, сгенерированный по алгоритму **RSA** с длиной ключа 4096 бит. Следующим шагом напишем конфигурационный файл `ca_req.cnf` для создания на его основе запроса на сертификат. 

```bash
vi ca_req.cnf
```

Он будет следующего содержания (*не забудьте изменить* значения полей `OU` и `CN`):

```conf
[ req ]
default_bits = 4096
encrypt_key = no
default_md = sha256
prompt = no
utf8 = yes
distinguished_name = ca_distinguished_name

[ ca_distinguished_name ]
C = RU
ST = Moscow State
L = Moscow
O = BMSTU
OU = IU4-XX
CN = Takumi Fujiwara
```

![](../images/lab_3/3.2.png)

После чего выполним команду для создания запроса на сертификат:

```bash
openssl req -new -key private/ca.key -out certs/ca.csr -config ca_req.cnf
```

![](../images/lab_3/3.3.png)

Если мы решим посмотреть на содержание файла с запросом, то увидим некий набор символов.

![](../images/lab_3/3.4.png)

Это следствие того, что сертификат по умолчанию хранится в формате **base64**. Однако есть возможность увидеть поля запроса и их значения, например, командой:

```bash
openssl req -text -noout -in certs/ca.csr
```

![](../images/lab_3/3.5.png)

Теперь можно заметить, что поля `Subject` совпадают со значениями этих полей в конфигурационном файле. Еще одно важное поле - `Modulus`, содержит открытый ключ.

Создадим еще один конфигурационный файл с именем `ca.cnf`, он будет описывать параметры работы нашего удостоверяющего центра.

```bash
vi ca.cnf
```

```conf
[ ca ]
default_ca = batman_ca

[ batman_ca ]
# Путь к каталогу CA
dir               = .
certs             = $dir/certs
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# Путь к файлам ключей и сертификатов CA
private_key       = $dir/private/ca.key
certificate       = $dir/certs/ca.crt

# Настройки выдачи сертификатов
copy_extensions   = copy
default_days      = 375
default_md        = sha256
preserve          = no
policy            = ca_policy

[ ca_policy ]
countryName = match
stateOrProvinceName = supplied
organizationName = supplied
commonName = supplied
organizationalUnitName = optional
commonName = supplied

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
```

![](../images/lab_3/3.6.png)

Ещё для корректной работы удостоверяющего центра потребуется создать файл с базой данных выпускаемых сертификатов:

```bash
touch index.txt
```

И случайный серийный номер:

```bash
openssl rand -hex 16 > serial
```

И проверим структуру `pki`

![](../images/lab_3/3.7.png)

Выполним команду для генерации самоподписанного сертификата УЦ:

```bash
openssl ca -selfsign -days 3650 -in certs/ca.csr -out certs/ca.crt -config ca.cnf -extensions v3_ca
```

![](../images/lab_3/3.8.png)

Результатом выполнения команд является сертификат удостоверяющего центра.

![](../images/lab_3/3.9.png)

Теперь займёмся сертификатом веб-сервера. Для начала создадим конфигурационный файл `webserver_req.cnf`. На данном этапе важно обратить внимание на значение поля `CN` (Common Name). В этом поле необходимо указать доменное имя или IP-адрес ВМ, иначе субъект проверки не будет соответствовать предъявляемому сертификату. В секции `alt_names` задают дополнительные доменные имена и IP-адреса.

```bash
vi webserver_req.cnf
```

```conf
[ req ]
default_bits = 2048
encrypt_key = no
default_md = sha256
prompt = no
utf8 = yes
distinguished_name = webserver_dn
req_extensions = webserver_ext

[ webserver_dn ]
C = RU
ST = Moscow State
L = Moscow
O = BMSTU
OU = IU4-XX
CN = 192.168.243.128

[ webserver_ext ]
subjectAltName = @alt_names

[alt_names]
IP.1 = 192.168.243.128
DNS.1 = takumi.fujiwara.local
```

![](../images/lab_3/3.10.png)

При помощи следующей команды создадим и закрытый ключ веб-сервера, и запрос на сертификат веб-сервера (да-да, одной командой сразу!):

```bash
openssl req -new -nodes -keyout webserver.key -out certs/webserver.csr -config webserver_req.cnf
```

![](../images/lab_3/3.11.png)

Далее следует выпустить сертификат веб-сервера, подписанный закрытым ключом удостоверяющего центра, сделать это можно при помощи команды:

```bash
openssl ca -in certs/webserver.csr -out certs/webserver.crt -config ca.cnf -extensions server_cert
```

![](../images/lab_3/3.12.png)

Полученный сертификат веб-сервера:

![](../images/lab_3/3.13.png)

Атрибут `X509v3 Authority Key Identifier` - идентификатор ключа УЦ, которым подписан сертификат. Другими словами, по этому полю можно определить, действительно ли сертификат подписан конкретным удостоверяющим центром. Чтобы это проверить, можно воспользоваться следующими командами:

```bash
grep -A 1 "Authority Key Identifier" certs/webserver.crt
```

```bash
grep -A 1 "Subject Key Identifier" certs/ca.crt
```

![](../images/lab_3/3.14.png)

Осталось отредактировать конфигурационный файл NGINX, где указать параметры TLS и пути к сертификатам. Предварительно создадим необходимые каталоги и файлы.

В `/etc/nginx/` создадим папку `ssl`. В ней создадим файл `dhparam.pem` - файл с ключевой информацией для поддержки технологии **Forward Secrecy**. Эта технология позволяет не допустить расшифровку трафика TLS-сессии даже при компрометации закрытого ключа сервера.

```bash
sudo mkdir /etc/nginx/ssl
```

Сгенерируем`dhparam.pem` следующей командой:

```bash
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 4096
```

![](../images/lab_3/3.15.png)

>Генерация может занять некоторое время, поэтому пока-что можете сходить перекусить.

![](../images/lab_3/pic%201.png)

Теперь скопируем в каталог `ssl` сертификат и закрытый ключ веб-сервера.

```bash
sudo cp ./pki/certs/webserver.crt ./pki/webserver.key /etc/nginx/ssl/
```

![](../images/lab_3/3.16.png)

### Настройка веб-сервера и хоста
Все настройки TLS было бы неплохо вынести в отдельный файл (пусть он называется `ssl_params`, а находится в корневом каталоге конфигураций NGINX). Это позволит его переиспользовать в случае настройки нескольких сайтов.

```bash
sudo vi /etc/nginx/ssl_params
```

Содержимое файла ([ссылка на референс](https://ssl-config.mozilla.org/)):

```nginx
ssl_certificate ssl/webserver.crt;
ssl_certificate_key ssl/webserver.key;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:10m;

ssl_dhparam ssl/dhparam.pem;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
```

![](../images/lab_3/3.17.png)

>[!NOTE]
>Директивы `ssl_certificate` и `ssl_certificate_key` указывают на расположение сертификата и закрытого ключа сервера, `ssl_protocols` - на используемые протоколы, `ssl_ciphers` – на наборы шифров, `ssl_dhparam` - путь к файлу `dhparam.pem`, сгенерированному немного ранее.

Наконец, внесем изменения в конфиг NGINX:

```bash
vi /etc/nginx/conf.d/flask_app.conf
```

```nginx
upstream flask_app {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}


server {
    listen 443 ssl;
    server_name _;
    include /etc/nginx/ssl_params;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://flask_app;
    }
}
```

После чего перезагрузим его.

```bash
sudo nginx -s reload
```

![](../images/lab_3/3.18.png)

>[!NOTE]
>Создан новый виртуальный сервер на порту 80, а порт имеющегося изменен на 443. Предварительно заполненный файл `ssl_params` с настройками TLS подключен директивой `include`, в которой указан абсолютный путь к этому файлу.

Теперь скачаем файл сертификата на хост для его последующей установки на оном.

![](../images/lab_3/3.19.png)

Откроем скачанный файл. В глаза бросается, что наша операционная система не доверяет данному сертификату.

![](../images/lab_3/3.20.png)

Во вкладке `Состав` и `Путь сертификации` можно посмотреть более подробную информацию о сертификате.

![](../images/lab_3/3.21.png)

![](../images/lab_3/3.22.png)

Для того, чтобы ОС доверяла нашему сертификату установим его в хранилище доверенных корневых сертификатов (кнопка `Установить сертификат...` во вкладке `Общие`):

![](../images/lab_3/3.23.png)

![](../images/lab_3/3.24.png)

![](../images/lab_3/3.25.png)

![](../images/lab_3/3.26.png)

![](../images/lab_3/3.27.png)

После установки еще раз откроем сертификат на хосте.

![](../images/lab_3/3.28.png)

Теперь ОС хоста будет доверять всем сертификатам, подписанным закрытым ключом нашего импровизированного удостоверяющего центра.

Проверим работу сайта по протоколу HTTPS:

![](../images/lab_3/3.29.png)

Видим значок, сигнализирующий о корректной проверке сертификата сайта браузером. Кликнув на него и выбрав соответствующий пункт в выпадающем меню, можно увидеть и сам сертификат.

>[!NOTE]
>Добавим пару слов про строчку в конифге NGINX `return 301 https://$host$request_uri;`. Смысл этой настройки в том, чтобы принудительно направить браузер на HTTPS версию сайта, даже если пользователь пытается открыть его по HTTP. В таком случае сервер отправляет клиенту ответ с кодом `301` (Moved Permanently) и новым адресом. Браузер запоминает этот ответ и в следующий раз возьмет этот ответ из кэша и откроет HTTPS версию быстрее.

Попытаемся намеренно открыть сайт по протоколу HTTP:

![](../images/lab_3/3.30.png)

Открыв в браузере консоль разработчика (клавиша `F12`) и обратившись к сайту по HTTP видим следующую картину:

![](../images/lab_3/3.31.png)

Если на хосте есть права администратора, можно провернуть еще один фокус. Откроем файл `C:\Windows\System32\drivers\etc\hosts` и добавим в него строку с IP-адресом виртуальной машины и доменным именем, которое было добавлено в сертификат.

![](../images/lab_3/3.32.png)

При попытке сохранить файл может появиться сообщение о том, что доступ запрещен. Следует нажать кнопку `Retry as Admin…`:

![](../images/lab_3/3.33.png)

Если все выполнено верно, в браузере можно открыть сайт не по IP-адресу, а по доменному имени. Браузер считает подключение валидным, потому что доменное имя добавлено в расширения сертификата:

![](../images/lab_3/3.34.png)

На данном этапе настроена инфраструктура открытых ключей, созданы сертификаты УЦ и веб-сервера, а сам веб-сервер настроен на работу по протоколу HTTPS.

---

## Работа с GIT
### Основы GIT
>[!NOTE]
>При разработке ПО удобно хранить код в системе контроля версий. Это позволяет контролировать процесс разработки, оперативно откатываться к предыдущим версиям кода, распараллеливать разработку между несколькими людьми, в случае хранения кода во внешнем репозитории - иметь доступ к актуальной версии ПО из любого места с доступом в интернет. В данных лабораторных работах в качестве системы контроля версий выступает GIT.

Проверить, установлен ли GIT, а заодно узнать его версию можно так:

```bash
git --version
```

![](../images/lab_3/3.35.png)

После установки необходимо выполнить настройки имени пользователя и почты для того, чтобы система могла указать правильные сведения о авторе изменений в файлах:

```bash
git config --global user.name "batman"
```

```bash
git config --global user.email "batman@gotham.com"
```

Эти и другие настройки могут быть записаны в файл `~/.gitconfig`. Посмотреть его содержимое можно командой:

```bash
git config --list
```

![](../images/lab_3/3.36.png)

Для дальнейшей работы настроим окружение в директории `usr/share/nginx/html/` где расположим наш GIT-репозиторий.

Сначала задаём нужные права доступа:

```bash
sudo chmod 775 /usr/share/nginx/html/
```

И устанавливаем владельца для корневой папки `html`.

```bash
sudo chown nginx:nginx /usr/share/nginx/html/
```

Затем добавляем текущего пользователя в группу `nginx`, чтобы он мог свободно работать с файлами.

```bash
sudo usermod -aG nginx $USER
```

Активируем это членство, обновив список групп пользователя:

```bash
newgrp nginx
```

![](../images/lab_3/3.37.png)

Создаём директорию `demo_project` для нового проекта и переходим в неё.

```bash
mkdir /usr/share/nginx/html/demo_project
```

```bash
cd /usr/share/nginx/html/demo_project
```

И теперь наконец можем инициализировать локальный GIT-репозиторий:

```bash
git init
```

![](../images/lab_3/3.38.png)

После инициализации репозитория появилась скрытая папка `.git`, в которой находятся служебные файлы, относящиеся к GIT:

![](../images/lab_3/3.39.png)

Проверить статус созданного репозитория можно командой:

```bash
git status
```

![](../images/lab_3/3.40.png)

>[!NOTE]
>Строка `On branch master` показывает, в какой ветке (branch) мы сейчас находимся. Строка `No commits yet` говорит о том, что пока еще нет ни одного коммита.

Создадим новый файл в нашем репозитории.

```bash
echo "# Description of the demonstration project" > README.md
```

Если мы снова посмотрим статус репозитория, то увидим, что созданный файл отображается, как `Untracked files`.

![](../images/lab_3/3.41.png)

Для того, чтобы добавить файл в отслеживаемые гитов введём команду:

```bash
git add README.md
```

Теперь в статусе написано, что файл README.md добавлен и готов к коммиту.

![](../images/lab_3/3.42.png)

Теперь мы можем сделать наш первый коммит, то есть зафиксировать изменения сделанные в нашем проекте.

```bash
git commit -m "Added a README file with a description of the project"
```

![](../images/lab_3/3.43.png)

>[!NOTE]
>В выводе команды отображена текущая ветка, номер нового коммита (на самом деле тут только первые 7 символов номера) и список изменений (файлов и строк кода). Номер коммита - это хэш по алгоритму SHA-1 файлов в проекте. То есть абсолютно одинаковые коммиты будут иметь одинаковые номера, что позволит GIT знать об их идентичности.

Теперь в статусе написано, что нет ничего, что можно закоммитить.

![](../images/lab_3/3.44.png)

Посмотрим историю коммитов.

```bash
git log
```

![](../images/lab_3/3.45.png)

>[!NOTE]
>Вывод команды отображает информацию о коммитах в репозитории:
>- его уникальный хеш — `3991063...`;
>- ветка — `master`;
>- автор и его мэйл — `batman <batman@gotbam.com>`;
>- дата и время создания;
>- комментарий коммита.

Теперь внесём изменения в наш файл `README.md`.

```bash
vi README.md
```

Текст должен выглядеть так:

```markdown
# Description of the demonstration project

This is a demo project created to gain skills in working with Git.

---

Batman
```

![](../images/lab_3/3.46.png)

И закоммитим наши изменения.

```bash
git add README.md
```

```bash
git commit -m "The README file has been modified"
```

![](../images/lab_3/3.47.png)

Теперь повторим эти действия, но удалим из файла `---`.

```bash
vi README.md
```

```markdown
# Description of the demonstration project

This is a demo project created to gain skills in working with Git.

Batman
```

```bash
git add README.md
```

```bash
git commit -m "The horizontal separator has been removed from the README file"
```

![](../images/lab_3/3.48.png)

Посмотрим историю наших коммитов.

```bash
git log
```

![](../images/lab_3/3.49.png)

Давайте попробуем откатить изменения в нашем `README.md` до того момента когда элемент `---` ещё не был удалён. Есть два варианта, как это сделать.

>[!NNOTE]
>`git revert` — создаёт новый коммит, который отменяет изменения указанного предыдущего коммита, сохраняя историю изменений.  
>
>`git reset`  — перемещает указатель ветки назад к указанному коммиту и может удалять коммиты из истории (в зависимости от режима: `--soft`, `--mixed`, `--hard`).

Давайте попробуем первый вариант указав после `git revert` идентификатор последнего коммита:

```bash
git revert b8f3546
```

После этого у нас выскочит окно, в котором будет автоматически сгенерированный комментарий для коммита генерируемого командой `git revert`, его можно изменить, но в данном случае оставим всё по умолчанию. Выйти отсюда можно так же как как из Vi.

![](../images/lab_3/3.50.png)

При просмотре истории мы можем увидеть новый коммит в котором мы находимся.

```bash
git log --oneline
```

![](../images/lab_3/3.51.png)

И содержание файла соответствует второму коммиту (в данном случае он под номером `9a14f0d`).

![](../images/lab_3/3.52.png)

Теперь воспользуемся вторым способом и вернёмся к коммиту `9a14f0d` удалив все коммиты следующие за ним.

```bash
git reset --hard HEAD~2
```

![](../images/lab_3/3.53.png)

Наполнение файла не изменилось, но теперь мы находимся в коммите `b5e8c8c`.

### Принципы использования веток
>[!NOTE]
>Ветки в Git — это независимые линии разработки, позволяющие работать над разными частями проекта параллельно, независимо друг от друга. Каждая ветка сохраняет свою историю изменений, что даёт возможность изолированно разрабатывать новые функции, исправлять ошибки или экспериментировать, а затем безопасно интегрировать результаты в основную ветку (например, `master`).
>
>На этом этапе лабораторной работы создаётся базовая HTML-структура страницы в ветке `html_base`, от которой ответвляются две новые ветки: `css` (для стилей CSS) и `script` (для скрипта на страничке). После внесения изменений в обе ветки `script` вливается обратно в `html_base`. Затем обновлённая `html_base` сливается с `master`, а вслед за ней — ветка `css`, что завершает сборку полной версии страницы.

Перед тем, как приступить к работе с ветками создадим еще один конфигурационный файл NGINX.

```bash
vi /etc/nginx/conf.d/demo_project.conf
```

 В него добавим конфигурацию сервера, который будет работать на порту 8080:

```nginx
server {
    listen 8080;
    server_name _;

    location / {
        root   /usr/share/nginx/html/demo_project/html;
        index  index.html;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

И еще добавим порт `8080` в файрволл и выключим SELinux.

```bash
sudo firewall-cmd --add-port=8080/tcp
```

```bash
sudo setenforce 0
```

И перезагрузим NGINX.

```bash
sudo nginx -s reload
```

![](../images/lab_3/3.54.png)

В начале посмотрим какие ветки есть. По умолчанию создаётся ветка `master` (как в нашем случае) или `main`.

```bash
git branch
```

![](../images/lab_3/3.55.png)

Создадим новую ветку `html_base`:

```bash
git checkout -b html_base
```

![](../images/lab_3/3.56.png)

И теперь при просмотре существующих веток видно `html_base`, при этом напротив стоит `*`, это значит, что мы находимся в этой ветке.

Создадим директорию `html`, а в ней файл `index.html`.

```bash
mkdir html && vi html/index.html
```

Наполнение `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Batman</title>
</head>
<body>
    <h1>Learning Through KTI Labs</h1>
    <p>Laboratory work helps turn theory into practical knowledge. Each experiment builds skills and deepens understanding.</p>
    <p>Prepare, observe, analyze, and reflect. This is how expertise grows.</p>
</body>
</html>
```

![](../images/lab_3/3.57.png)

Закоммитим изменения.

```bash
git add html/index.html
```

```bash
git commit -m "Added HTML page base"
```

![](../images/lab_3/3.58.png)

Теперь мы может открыть эту страничку в браузере:

![](../images/lab_3/3.59.png)

Из ветки `html_base` сделаем 2 новых.

```bash
git branch css && git branch script
```

И перейдём в `script`:

```bash
git checkout script
```

![](../images/lab_3/3.60.png)

И добавим в конец `body` скриптик:

```bash
vi html/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Batman</title>
</head>
<body>
    <h1>Learning Through KTI Labs</h1>
    <p>Laboratory work helps turn theory into practical knowledge. Each experiment builds skills and deepens understanding.</p>
    <p>Prepare, observe, analyze, and reflect. This is how expertise grows.</p>
    
    <script>
        const encrypted =
            "Ly8gRm9sbG93IHRoZSB3aGl0ZSByYWJiaXQuLi4KLy8gU3lzdGVtIG" +
            "luaXRpYWxpemVkLi4uCi8vIEtub3dsZWRnZTogOTcuMyUKLy8gV2VsY2" +
            "9tZSB0byB0aGUgcmVhbCB3b3JsZC4KLy8gVGhlcmUgaXMgbm8gc3Bvb24u";

        let seq = '';
        document.addEventListener('keydown', e => {
            seq = (seq + e.key.toUpperCase()).slice(-6);
            if (seq === 'MATRIX') {
                document.getElementById('matrixCode').textContent = atob(encrypted);
                document.getElementById('matrixCode').style.display = 'block';
                setTimeout(() => {
                    document.getElementById('matrixCode').style.display = 'none';
                }, 8000);
            }
        });
    </script>
</body>
</html>
```

![](../images/lab_3/3.61.png)

Закоммитим.

```bash
git add html/index.html
```

```bash
git commit -m "JavaScript script added"
```

![](../images/lab_3/3.62.png)

Теперь перейдём в `css`:

```bash
git checkout css
```

Можно заметить, что здесь нет изменений, которые мы внесли в ветке `script`.

![](../images/lab_3/3.63.png)

Добавим в `head` описание стилей:

```bash
vi html/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Batman</title>
    <style>
        body {
            font-family: 'Segoe UI', system-ui, sans-serif;
            max-width: 600px;
            margin: 40px auto;
            padding: 0 20px;
            line-height: 1.6;
            color: #333;
            background-color: #fafafa;
        }
        h1 {
            text-align: center;
            margin-bottom: 24px;
            color: #2c3e50;
            font-weight: 500;
        }
        p {
            margin-bottom: 16px;
            color: #555;
        }
        .matrix {
            display: none;
            font-family: monospace;
            color: #0f0;
            background: #000;
            padding: 16px;
            margin-top: 20px;
            border-radius: 4px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.1);
        }
    </style>
</head>
<body>
    <h1>Learning Through KTI Labs</h1>
    <p>Laboratory work helps turn theory into practical knowledge. Each experiment builds skills and deepens understanding.</p>
    <p>Prepare, observe, analyze, and reflect. This is how expertise grows.</p>
</body>
</html>
```

![](../images/lab_3/3.64.png)

Снова коммитим.

```bash
git add html/index.html
```

```bash
git commit -m "Added CSS styles"
```

![](../images/lab_3/3.65.png)

Если посмотреть нашу страничку в браузере, то можно увидеть что её внешний вид изменился.

![](../images/lab_3/3.66.png)

Теперь вернёмся в ветку `html_base`:

```bash
git checkout html_base
```

![](../images/lab_3/3.67.png)

При этом в браузере страничка откатиться к первоначальному состоянию.

![](../images/lab_3/3.68.png)

Внесём изменения в HTML код:

```bash
vi html/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Batman</title>
</head>
<body>
    <h1>Learning Through KTI Labs</h1>
    <p>Laboratory work helps turn theory into practical knowledge. Each experiment builds skills and deepens understanding.</p>
    <p>Prepare, observe, analyze, and reflect. This is how expertise grows.</p>
    
    <div class="matrix" id="matrixCode"></div>
</body>
</html>
```

![](../images/lab_3/3.69.png)

И закоммитим изменения.

```bash
git add html/index.html
```

```bash
git commit -m "Added div element"
```

![](../images/lab_3/3.70.png)

Попробуем слить ветку `script` в `html_base`, для этого находясь в ветке `html_base` пропишем:

```bash
git merge script
```

![](../images/lab_3/3.71.png)

Нам выдало ошибку, потому что строка `<div class="matrix" id="matrixCode"></div>` которую мы добавили `html_base` находится там же где открывающий тег `<script>` в ветке `script`. Из-за этого возникает исключение, так как GIT не понимает, какую строчку нужно оставить, а какую нет. Поэтому он возлагает эту задачу на нас.

Откроем файл с помощью текстового редактора.

```bash
vi html/index.html
```

![](../images/lab_3/3.72.png)

Можем заметить появившиеся строки с `<<<<<<< HEAD`, `=======` и `>>>>>> script`. Это автоматически генерируемые GIT строки указывающие на область конфликта. Удалим их и приведём файл к состоянию как на скриншоте ниже:

![](../images/lab_3/3.73.png)

После чего коммитим, и на этом слияние закончено.

```bash
git add html/index.html
```

```bash
git commit
```

![](../images/lab_3/3.74.png)

Переключимся в ветку `master` и вольём в неё ветку `html_base`, а потом ветку `css`.

```bash
git checkout master
```

```bash
git merge html_base
```

```bash
git merge css
```

![](../images/lab_3/3.75.png)

И в итоге всех наших действий получаем целостный файл с кодом странички:

![](../images/lab_3/3.76.png)

Сама страничка будет выглядеть так:

![](../images/lab_3/3.77.png)

Мы можем посмотреть структуру веток при помощи команды:

```bash
git log --oneline --graph
```

![](../images/lab_3/3.78.png)

Это может выглядеть не очень наглядно, поэтому мы можем установить расширение:

![](../images/lab_3/3.79.png)

После чего можем посмотреть на структуру веток с красивым интерфейсом.

![](../images/lab_3/3.80.png)

### Использование GitHub
>[!NOTE]
>[GitHub](github.com) — это облачная платформа для хостинга GIT-репозиториев, которая позволяет разработчикам не только хранить и резервировать код, но и эффективно сотрудничать при работе над общими проектами. С помощью GitHub можно делиться кодом с другими, отслеживать изменения, обсуждать задачи, проверять предложения по улучшению через **Pull Request** и автоматизировать процессы сборки и тестирования.

Зайдём на страничку [GitHub](github.com) и перейдём в наш аккаунт, если такого у вас нет, то надо его создать, нажав на `Sign up`.

![](../images/lab_3/3.81.png)

>Интересный факт: эти методички тоже хранятся на GitHub, поэтому вы можете скачать их на свои компьютеры с помощью GIT и хранить локально, главное периодически их обновлять. А ещё вы можете поставить звёздочку на этот репозиторий с методами :)

Откроем наш профиль и перейдём во вкладку `Repositories`.

![](../images/lab_3/3.82.png)

Далее нажмем на кнопку `New`.

![](../images/lab_3/3.83.png)

Создаём публичный репозиторий с названием `KTI_demo_project`:

![](../images/lab_3/3.84.png)

После этого нас перенаправит на страницу репозитория. Нажмем на `SSH`. И GitHab любезно предоставит нам команды для подключения нашего локального репозитория к удалённому.

![](../images/lab_3/3.85.png)

Теперь в консоли нашей ВМ введём:

```bash
git remote add origin git@github.com:Porfik/KTI_demo_project.git
```

Проверим добавление в ваш локальный GIT-репозиторий удалённого репозитория.

```bash
git remote -v
```

А дальше переименовываем ветку в которой мы находимся (`master`) в `main`. Это связано с тем, что в GItHub автоматически создаётся ветка `main` и во избежание конфликтов при синхронизации мы переименовываем нашу главную ветку в локальном репозитории:

```bash
git branch -M main
```

![](../images/lab_3/3.86.png)

Для того чтобы взаимодействовать с GitHub нам нужно, чтобы он мог нас идентифицировать. Будем использовать для этого SSH ключи. Сгенерируем их:

```bash
ssh-keygen -t ed25519
```

![](../images/lab_3/3.87.png)

Скопируем наполнение `id_ed25519.pub`

![](../images/lab_3/3.88.png)

Перейдём в настройки аккаунта GitHub и выберем вкладку `SSH and GPG keys`. 

![](../images/lab_3/3.89.png)

И добавляем новый SSH ключ.

![](../images/lab_3/3.90.png)

![](../images/lab_3/3.91.png)

После этого мы наконец-то можем загрузить наш репозиторий на GitHub.

```bash
git push -u origin main
```

![](../images/lab_3/3.92.png)

Как видим, всё успешно загрузилось.

![](../images/lab_3/3.93.png)

Инструментарий GitHub позволяет вносить изменения в файлах прямо через веб интерфейс. Нажмём на значок карандаша:

![](../images/lab_3/3.94.png)

Внесём какие-нибудь изменения в файл README.md и закоммитим их нажав на соответствующую кнопку.

![](../images/lab_3/3.95.png)

![](../images/lab_3/3.96.png)

![](../images/lab_3/3.97.png)

Теперь синхронизируем изменения на GitHub с нашим локальным репозиторием:

```bash
git pull
```

![](../images/lab_3/3.98.png)

Как видим, изменения синхронизировались.

![](../images/lab_3/3.99.png)

После выполнения этой работы выпущен сертификат, настроен доступ к сайту по протоколу HTTPS, получены базовые навыки работы с GIT и GitHub.