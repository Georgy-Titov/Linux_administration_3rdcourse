## Задание 1

Развернуть с нуля корпоративный веб-сервер на своем домене в закрытом периметре. Получить SSL сертификат Let’s encrypt.

### Этапы выполнения.

1.	Установить Debian 13 на виртуальную машину. Можно использовать собственный VPS. Сетевой адатер виртуальной машины перевести в режим Bridge. Сконфигурировать статический IP адрес в своей домашней подсети. Проверить ping из виртуальной машины и с хостовой машины. Настроить временный ресолвинг домена `<abc>.lab-itmo.ru` в файле hosts на рабочем десктопе. Проверить через ping `<abc>.lab-itmo.ru`. Установить ssh и fail2ban. Настроить доступ root по ssh без пароля, далее работа по ssh.

2.	Установить веб-сервер nginx, сконфигурировать http вебсайт `<abc>.lab-itmo.ru`. Настроить  порт 8080 и отдельный журнал с именем домена. Создать индексную страницу с:

* ФИО студента
* Имя виртуального хоста из внутренней переменной nginx
* Дата создания

3.	Установить certbot. Получить SSL сертификат Let’s Encrypt для домена. Для этого нужно подтвердить владение доменом через DNS челлендж. Сконфигурировать SSL в nginx с полученым сертификатом на домене `<abc>.lab-itmo.ru` и стандартном порту 443.

> Данные для управления записями DNS регистратора Beget: login – avdoro57 | password - ITMO2026LabWork

### Дополнительные баллы: 

* получить сертификат в автоматическом режиме через авторизационный скрипт доменного регистратора
* получить wildcard сертификат *.lab-itmo.ru там же
* создать DNS A запись <abc>.lab-itmo.ru  для своего статического адреса, используя curl api.beget.com
* Сконфигурировать редирект nginx с http на https

---

### Установка Debian 13.x.x и настройка сети

С сайта (debain.org/download)[https://www.debian.org/download] скачаем iso Образ Debian 13.5.0. Поднимем ВМ в VirualBox и в настройках сети виртуальной мащины прокинем Сетвой мост.

<img width="1920" height="1037" alt="image" src="https://github.com/user-attachments/assets/35cb7e60-ea99-44b2-ba5e-e22a7eadc9f3" />

В самой виртуальной мащине переходим по пути `/etc/systemd/network/` и создаем конфиг файл для сетевого интерфейса enp0s8:

```
[Match]
Name=enp0s8

[Network]
Address=192.168.0.111/24
Gateway=192.168.0.1
DNS=8.8.8.8
DNS=1.1.1.1
```

Далее командой `ip -a` проверяем конфигурацию сетевых интерфейсов и на рабочем хосте пропингуем виртуальную машину:

<img width="1280" height="800" alt="image" src="https://github.com/user-attachments/assets/d43355b8-ed4e-40e7-b196-0e1f2d96a9f1" />

### Настройка временного резолва домена

В файле `/etc/hosts` добавляем запись о нашем домене и проверяем командой `ping tgk.lab-itmo.ru`:

<img width="1111" height="586" alt="image" src="https://github.com/user-attachments/assets/3dd15509-b68d-4c94-a649-dec58f9b4cc8" />

### Настройка SSH

На домашнем хосте надо сгенерировать пару ключей и закинуть публичный прокинуть публичный ключ ручками либо командой `ssh-copy-id root@<host ip>` на виртуальную машину в файл `/root/.ssh/authorized_keys`.

Далее открываем `/etc/ssh/sshd_config` и нас указываем следующие параметры:

```
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
```

Рестартим ssh сервис командой `systemctl restart ssh` и пытаемся зайти на виртуальную машину под пользователем `root` - `ssh root@192.168.0.111`

### Установка и настройка Nginx

Устанавливаем Nginx командой `apt-get install nginx -y` и запускаем сервис Nginx `systemctl start nginx`.

По пути `/var/www/tgk.lab-itmo.ru` создем морду нашего сайт `index.html`:

```
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>ITMO Linux Administration</title>
</head>
<body>
<h1>Georgy Titov</h1>
<p>Host: <!--#echo var="hostname" --></p>
<p>Дата создания: 12.06.2026</p>
</body>
</html>
```
Создаем конфигурацию по пути `/etc/nginx/sites-available/tgk.lab-itmo.ru`:

```
server {
    listen 8080;
    server_name tgk.lab-itmo.ru;

    root /var/www/tgk.lab-itmo.ru;
    index index.html;

    ssi on;

    access_log /var/log/nginx/tgk.lab-itmo.ru.access.log;
    error_log  /var/log/nginx/tgk.lab-itmo.ru.error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location = /host {
        return 200 "$hostname\n";
    }
}
```

Подключаем наш сайт `ln -s /etc/nginx/sites-available/tgk.lab-itmo.ru /etc/nginx/sites-enabled/`

Чтобы все зараотало - `nginx -t` - `systemctl restart nginx`

### Получение SSL-сертификата

1. `certbot certonly \ --manual \ --preferred-challenges dns \ -d tgk.lab-itmo.ru` - для того чтобы получить сертификат выполняем данную команду. После выполнения в консоли выведется справочная информация и токе. Он нам и понадобиться дальше. 

2. В соседнем терминале надо исполнить команду `curl "https://api.beget.com/api/dns/changeRecords?login=<<login>>&passwd=<<pass>>&input_format=json&output_format=json&input_data=%7B%22fqdn%22%3A%22_acme-challenge.andrz.lab-itmo.ru%22%2C%22records%22%3A%7B%22TXT%22%3A%5B%7B%22priority%22%3A10%2C%22value%22%3A%22<<token>>%22%7D%5D%7D%7D"`.

* **login** и **pass** берем из задания лабораторной работы, **token** получаем из команды выше.

Выполнив команду 2 надо подождать секунд 15-30 и выполнить в терминале, где выполняли команду 1 нажать `Enter`.

<img width="1034" height="521" alt="image" src="https://github.com/user-attachments/assets/f735036f-935e-434b-a401-79b7466f247d" />

Получившиеся сертификаты сохранились по пути `/etc/letsencrypt/live/tgk.lab-itmo.ru/`

```
root@debian:/etc/letsencrypt/live/tgk.lab-itmo.ru# ls -l
итого 4
lrwxrwxrwx 1 root root  39 июн 12 19:46 cert.pem -> ../../archive/tgk.lab-itmo.ru/cert1.pem                    # сертификат сервера
lrwxrwxrwx 1 root root  40 июн 12 19:46 chain.pem -> ../../archive/tgk.lab-itmo.ru/chain1.pem                  # цепочка промежуточных сертификатов
lrwxrwxrwx 1 root root  44 июн 12 19:46 fullchain.pem -> ../../archive/tgk.lab-itmo.ru/fullchain1.pem          # полный сертификат вместе с цепочкой доверия
lrwxrwxrwx 1 root root  42 июн 12 19:46 privkey.pem -> ../../archive/tgk.lab-itmo.ru/privkey1.pem              # закрытый ключ сервера
-rw-r--r-- 1 root root 692 июн 12 19:46 README
```

Проверить действительность сертификата можно командой `openssl x509 -in /etc/letsencrypt/live/tgk.lab-itmo.ru/fullchain.pem -text -noout -`:

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            06:a3:14:35:09:5e:7d:49:95:d9:80:88:4d:c8:69:ca:c8:ff
        Signature Algorithm: ecdsa-with-SHA384
        Issuer: C=US, O=Let's Encrypt, CN=YE1
        Validity
            Not Before: Jun 12 15:47:43 2026 GMT
            Not After : Sep 10 15:47:42 2026 GMT
        Subject: CN=tgk.lab-itmo.ru
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:e3:ce:69:59:4c:bc:84:54:0a:30:0f:38:31:57:
                    77:57:17:bf:12:28:5d:b7:93:74:b3:0e:c1:fe:b1:
                    a7:f5:ea:01:b5:7d:fd:80:68:bb:ea:05:c0:74:f5:
                    22:17:52:c8:46:f3:41:79:c0:9d:90:77:60:0c:e8:
...
```

После получения сертификата настроем защищенное соединение на веб-сервере Nginx. Отредактируем наш файл с конфирурацие следующим образом:

```
server {
    listen 80;
    server_name tgk.lab-itmo.ru;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name tgk.lab-itmo.ru;

    root /var/www/html;
    index index.html;

    ssi on;

    ssl_certificate /etc/letsencrypt/live/tgk.lab-itmo.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tgk.lab-itmo.ru/privkey.pem;

    access_log /var/log/nginx/tgk.lab-itmo.ru.access.log;
    error_log /var/log/nginx/tgk.lab-itmo.ru.error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location = /host {
        default_type text/plain;
        return 200 "$hostname\n";
    }
}
```

В первом серверном блоке прописали редирект, чтобы все запросы, которые идет по `http://`  перенаправлялись на -> `https://`.

Проверяем командой `curl`:

<img width="779" height="201" alt="image" src="https://github.com/user-attachments/assets/03936309-13aa-4e10-9ae8-74f742cb8594" />

Также проверяем доступность сервиса в браузере:

<img width="1280" height="810" alt="image" src="https://github.com/user-attachments/assets/fca05c9a-895c-412f-a201-172e995a103f" />

### Отчетные результаты

> Скриншот браузера с http сервисом и вашей индексной страницей на порту 8080 после шага 2

<img width="785" height="361" alt="image" src="https://github.com/user-attachments/assets/9b96025b-6b4e-4e46-8ea0-a06efb237e80" />

> Конфигурационный файл /etc/nginx/sites-available/<abc>.lab-itmo.ru

<img width="627" height="457" alt="image" src="https://github.com/user-attachments/assets/50d78fd8-5c9d-42a5-83ac-b720cee105a0" />

> Скриншот команды ss -4tunl

<img width="1078" height="162" alt="image" src="https://github.com/user-attachments/assets/6d5a653d-5476-4b0c-8fff-23b01b07a8a2" />

> Скриншот браузера с информацией о SSL сертификате>

<img width="1280" height="810" alt="image" src="https://github.com/user-attachments/assets/1e4060e0-7d81-40d1-9398-d5ee39f95224" />

> Файлы журналов nginx – access.log и error.log

<img width="1119" height="252" alt="image" src="https://github.com/user-attachments/assets/7394ced1-7404-4b4f-8d7a-26039b83a3ac" />


> Файлы сертификата и закрытого ключа Let’s Encrypt>

<img width="881" height="232" alt="image" src="https://github.com/user-attachments/assets/2fe236af-6e6b-4f00-bea2-e31a03aebc86" />

## Задание 2

Написать сервис (bash script) для получения API токена для ИИ-модели GigaChat. Настроить таймер для запуска скрипта каждые 20 минут. Сконфигурировать nginx как прокси для перенаправления запросов на внешние модели. Создать веб-страницы для работы с моделями.

> Обязательное условие: токен и API KEY должны храниться в файле с атрибутом 600 и владельцем root.

### Этапы выполнения.

1. Установить корневой сертификат Минцифры (https://www.gosuslugi.ru/crt)[https://www.gosuslugi.ru/crt] в локальное хранилище Debian. Проверить получение токена через `curl -X POST "https://ngw.devices.sberbank.ru:9443/api/v2/oauth"`. 

2.	Создать файл `/etc/secrets/gigachat.key` с переменной `AUTH_KEY` равной корпоративному ключу авторизации Сбер и атрибутом 600. Создать папку `/usr/local/etc/nginx/`, в ней создать файл `<abc>-lm-studio.conf` с личным токеном студента в формате переменной nginx и атрибутом 600. 

3.	Создать bash скрипт c именем `/usr/local/bin/token-script-<abc>.sh` и атрибутом 700, состоящий из:
* Двух переменных – имя конфигурационного файла nginx и имя временного файла
*	запроса токена через curl и вывод результата во временный файл, заголовок авторизации использует переменную $AUTH_KEY
*	парсера  временного файла json и получения токена c испольованием jq
*	записи токена в конфигурационный файл в формате переменной nginx в папку /usr/local/etc/nginx/ с атрибутом 600
*	перезапуска nginx

Проверить работу скрипта – после запуска должен появляться конфигурационный файл nginx, содержащий токен.

4.	Создать unit-файл `get-token-<abc>.service` в `/etc/systemd/system/`. Параметры: Type – однократный запуск, ExecStart – имя файла bash скрипта, EnvironmentFile – `/etc/secrets/gigachat.key` 

5.	Создать таймер для этого unit файла. Задержка после загрузки 2 секунды, при работе интервал запуска 20 минут. Обновить базу сервисов systemd, включить таймер и проверить статус сервиса

6.	Включить папку `/usr/local/etc/nginx/` в блок server {} вашей SSL конфигурации `/etc/nginx/sites-available/<abc>.lab-itmo.ru` 

7.	Сконфигурировать два блока location{} с прозрачным прокси. Первый блок проксирует запросы /api/chat на сервер `https://gigachat.devices.sberbank.ru/api/v1/chat/completions` и использует токен Сбер как переменную из включенной папки выше. Второй блок проксирует запросы /V1 на сервер `https://ai.alex-doron.com` и использует переменную вашего личного токена студента для авторизации. Таймаут ответа прокси 600 секунд.

8.	Написать HTML страницы для обеих LLM. В них нужно получить названия доступных моделей и сделать минимальное окно чат-бота.


### Дополнительные баллы: 

*	Добавить обработку ошибок
*	Добавить различные подходящие опции в unit-файлы
*	Добавить форматирование HTML страниц, шрифты, стили
*	Включить стриминг ответа модели в реальном времени
*	Вывести остаток токенов Сбер
*	Использовать другие методы работы с моделью, кроме чат-бота. Например, голосовой ввод или ввод изображения, получить/загрузить файл, и т.п.
*	Настроить кастомный журнал прокси с логированием текста запроса, времени ответа, числа потраченых токенов
*	Настроить число повторения попыток запроса при ошибках или таймаутах


---
