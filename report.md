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

1. Установить корневой сертификат Минцифры [https://www.gosuslugi.ru/crt](https://www.gosuslugi.ru/crt) в локальное хранилище Debian. Проверить получение токена через `curl -X POST "https://ngw.devices.sberbank.ru:9443/api/v2/oauth"`. 

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

### Установка корневых сертификатов Минцифры

По инструкции на сайте [https://www.gosuslugi.ru/crt](https://www.gosuslugi.ru/crt) скачиваем сертификаты, распаковываем их и переносим в `/usr/local/share/ca-certificates/` и выполяем команду `update-ca-certificates`.

### Создание секретного ключа

Переходим на (developers.sber.ru)[https://developers.sber.ru/dev], регистрируемся, переходим в профиль, создаем новый провет GigaChat и для доступа к API генерируем Authorization Key.

Полученный ключ сохраним в виде переменной в файле `/etc/secrets/gigachat.key`:

```
chmod 600 /etc/secrets/gigachat.key
chown root:root /etc/secrets/gigachat.key
```

### Скрипт генерации api-токена

По пути `/var/local/bin/` создаем скрипт `generate-token.sh` 

```
#!/bin/bash

# Путь к nginx-конфигурации, в которую будет записан токен GigaChat
TOKEN_FILE="/usr/local/etc/nginx/gigachat.conf"

# Временный файл для сохранения ответа API в формате JSON
TMP_FILE="/tmp/gigachat_token.json"

# Подключение файла с переменной AUTH_KEY
source /etc/secrets/gigachat.key

# Генерация уникального идентификатора запроса
UUID=$(uuidgen)

# Запрос нового токена доступа к API GigaChat
curl -s -X POST \
  "https://ngw.devices.sberbank.ru:9443/api/v2/oauth" \
  -H "Authorization: Basic ${AUTH_KEY}" \
  -H "RqUID: ${UUID}" \
  -H "Accept: application/json" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "scope=GIGACHAT_API_PERS" \
  > "${TMP_FILE}"

# Вывод полученного JSON в консоль для контроля
cat "${TMP_FILE}"

# Извлечение поля access_token из JSON с помощью jq
TOKEN=$(jq -r '.access_token' "${TMP_FILE}")

# Проверка успешности получения токена
if [ -z "${TOKEN}" ] || [ "${TOKEN}" = "null" ]; then
    echo "ERROR: token not received"
    exit 1
fi

# Создание nginx-переменной с новым токеном
cat > "${TOKEN_FILE}" <<EOF
set \$gigachat_token "${TOKEN}";
EOF

# Назначение владельца root
chown root:root "${TOKEN_FILE}"

# Установка прав доступа 600
chmod 600 "${TOKEN_FILE}"

# Проверка корректности конфигурации nginx
# и применение новой конфигурации без остановки сервера
nginx -t && systemctl reload nginx

# Удаление временного файла
rm -f "${TMP_FILE}"
```

### Systemd service 



```
[Unit]
Description=Get SberBank GigaChat Token
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
EnvironmentFile=/etc/secrets/gigachat.key
ExecStart=/usr/local/bin/generate-token.sh
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

### Systemd timer

```
[Unit]
Description=Run GigaChat token update every 20 minutes

[Timer]
OnBootSec=2sec
OnUnitActiveSec=20min
Unit=get-token-gigachat.service

[Install]
WantedBy=timers.target
```

---

```
# Для того чтобы все заработало выполяем команды:
systemctl daemon-reload
systemctl enable --now get-token-andrz.timer
```

### Настройка Nginx

В конфигурацию нашего сайта добавляем пару новых `location` и `include`:

```
...

   include /usr/local/etc/nginx/*.conf;

...

   location /api/chat {
        proxy_pass https://gigachat.devices.sberbank.ru/api/v1/chat/completions;
        proxy_ssl_server_name on;

        proxy_set_header Authorization "Bearer $gigachat_token";
        proxy_set_header Content-Type "application/json";
        proxy_buffering off;
        chunked_transfer_encoding on;
        proxy_read_timeout 600s;
        proxy_connect_timeout 600s;
   }

    location /api/models {
       proxy_pass https://gigachat.devices.sberbank.ru/api/v1/models;

       proxy_ssl_server_name on;

       proxy_set_header Authorization "Bearer $gigachat_token";
       proxy_set_header Content-Type "application/json";

       proxy_buffering off;
   }
```

### WEB-морда

Для взаимодействия с моделью GigaChat была разработана HTML-страница Gigachat Proxy, предоставляющая минималистичный веб-интерфейс для отправки запросов к модели через настроенный nginx-прокси.

Основные возможности

* Получение списка доступных моделей через endpoint /api/models.
* Выбор модели из выпадающего списка.
* Ввод пользовательского запроса в текстовое поле.
* Отправка запроса к модели через endpoint /api/chat.
* Поддержка потоковой передачи ответа (Streaming), что позволяет отображать текст по мере его генерации моделью.
* Вывод статуса выполнения операций (загрузка моделей, отправка запроса, завершение генерации, ошибки).
* Обработка ошибок при обращении к API.

```
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Gigachat Proxy</title>

...

</script>

</body>
</html>
```


### Отчетные результаты

> Скриншоты вывода команды curl -Iv https://ngw.devices.sberbank.ru:9443

<img width="921" height="692" alt="image" src="https://github.com/user-attachments/assets/07a27a9a-70ee-4d4f-b292-a4e506e84c63" />

> Содержимое папки /usr/local/etc/nginx/ - файл с токеном Сбер и файл с личным токеном студента

> <img width="1240" height="199" alt="image" src="https://github.com/user-attachments/assets/ea6bbc3b-6973-4d8c-aa0c-6d200cb82aef" />

> Файл скрипта usr/local/bin/token-script-<abc>.sh

<img width="549" height="597" alt="image" src="https://github.com/user-attachments/assets/237d2578-497e-4d12-aaca-b91c6228bb3c" />

>	Unit-файлы get-token-<abc>.service и get-token-<abc>.timer

<img width="628" height="486" alt="image" src="https://github.com/user-attachments/assets/bb972a10-c263-4371-ab35-c28aa2540fcf" />

> Скриншот вывода systemctl status get-token-<abc>

<img width="1123" height="291" alt="image" src="https://github.com/user-attachments/assets/1ae7b104-ca15-4bb5-a0e9-bf552a1c6ac1" />

> Скриншот вывода systemctl list-timers

<img width="1146" height="299" alt="image" src="https://github.com/user-attachments/assets/a64869f9-a3f9-4c05-8675-f08ea2d6a948" />

> Конфигурационный файл /etc/nginx/sites-available/<abc>.lab-itmo.ru

<img width="839" height="897" alt="image" src="https://github.com/user-attachments/assets/7c905eb8-69ab-480e-a183-ba2ad9defb60" />

> Файлы HTML страницы для работы с моделями

```
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Gigachat Proxy</title>

<style>
:root {
    --bg: #0b0c10;
    --panel: #15171c;
    --panel2: #1c1f26;
    --text: #e6e6e6;
    --muted: #a1a1aa;
    --accent: #f97316;
    --border: #2a2d36;
}

body {
    margin: 0;
    font-family: Arial, sans-serif;
    background: var(--bg);
    color: var(--text);
}

.container {
    max-width: 900px;
    margin: 40px auto;
    padding: 24px;
    background: var(--panel);
    border-radius: 12px;
    box-shadow: 0 0 30px rgba(0,0,0,0.5);
}

h1 {
    text-align: center;
    color: var(--accent);
    margin-bottom: 20px;
}

select, textarea {
    width: 100%;
    box-sizing: border-box;
    margin-top: 10px;
    padding: 12px;
    border-radius: 10px;
    border: 1px solid var(--border);
    background: var(--panel2);
    color: var(--text);
    outline: none;
}

textarea {
    height: 140px;
    resize: vertical;
}

button {
    background: var(--accent);
    border: none;
    color: #111;
    padding: 10px 14px;
    margin-top: 12px;
    border-radius: 10px;
    cursor: pointer;
    font-weight: 600;
    transition: 0.2s;
}

button:hover {
    opacity: 0.9;
}

pre {
    background: #0a0b0f;
    padding: 15px;
    border-radius: 10px;
    margin-top: 15px;
    white-space: pre-wrap;
    border: 1px solid var(--border);
    min-height: 140px;
}

.status {
    font-size: 12px;
    color: var(--muted);
    margin-left: 10px;
    transition: opacity 0.5s ease;
}
</style>
</head>

<body>

<div class="container">

<h1>Gigachat Proxy</h1>

<button onclick="loadModels()">Get Models</button>
<span class="status" id="status"></span>

<select id="models"></select>

<textarea id="prompt" placeholder="Enter your request..."></textarea>

<button onclick="ask()">Send</button>

<pre id="answer">Response will appear here...</pre>

</div>

<script>

function setStatus(msg, timeout = 3000) {
    const el = document.getElementById("status");
    el.innerText = msg;
    el.style.opacity = "1";

    setTimeout(() => {
        el.style.opacity = "0";
    }, timeout);
}

async function loadModels() {
    try {
        setStatus("Loading models...");

        const response = await fetch("/api/models");
        if (!response.ok) throw new Error("HTTP " + response.status);

        const data = await response.json();

        let models = document.getElementById("models");
        models.innerHTML = "";

        if (!data.data) throw new Error("Bad API response");

        data.data.forEach(m => {
            let opt = document.createElement("option");
            opt.value = m.id;
            opt.text = m.id;
            models.appendChild(opt);
        });

        setStatus("Models loaded");

    } catch (e) {
        setStatus("Error loading models");
        document.getElementById("answer").innerText = e.message;
    }
}

async function sendRequest(payload) {
    return fetch("/api/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload)
    });
}

async function ask() {
    try {
        setStatus("Streaming...");

        const model = document.getElementById("models").value;
        const text = document.getElementById("prompt").value;

        const response = await sendRequest({
            model,
            stream: true,
            messages: [{ role: "user", content: text }]
        });

        if (!response.ok) throw new Error("HTTP " + response.status);

        const reader = response.body.getReader();
        const decoder = new TextDecoder("utf-8");

        let result = "";
        document.getElementById("answer").innerText = "";

        while (true) {
            const { value, done } = await reader.read();
            if (done) break;

            const chunk = decoder.decode(value, { stream: true });

            for (let line of chunk.split("\n")) {
                if (!line.startsWith("data:")) continue;

                let jsonStr = line.replace("data:", "").trim();

                if (jsonStr === "[DONE]") {
                    setStatus("Done");
                    return;
                }

                try {
                    let data = JSON.parse(jsonStr);
                    let token = data.choices?.[0]?.delta?.content;

                    if (token) {
                        result += token;
                        document.getElementById("answer").innerText = result;
                    }
                } catch {}
            }
        }

        setStatus("Done");

    } catch (e) {
        setStatus("Error");
        document.getElementById("answer").innerText = e.message;
    }
}

</script>

</body>
</html>
```

> Скриншоты браузера с выводом списка доступных моделей

<img width="1280" height="802" alt="image" src="https://github.com/user-attachments/assets/9b2f8c13-f3a3-489b-98f5-79852ac4202d" />


> Скриншоты браузера с ответами чат-бота на запрос «В чем разница обучения студентов с преподавателем или без» в обеих моделях

<img width="1276" height="808" alt="image" src="https://github.com/user-attachments/assets/d7947367-0bfb-4cf7-baad-3b19c034bc1a" />

## Выводы

В ходе выполнения лабораторной работы были получены практические навыки администрирования Linux-серверов и развертывания веб-сервисов в корпоративной инфраструктуре.

В первой части работы была выполнена установка и базовая настройка операционной системы Debian, настроена статическая IP-адресация, удалённый доступ по SSH и защита сервера с использованием Fail2Ban. Был развернут веб-сервер Nginx, настроен виртуальный хост на собственном домене, организовано ведение журналов доступа и ошибок, а также получен и установлен SSL-сертификат Let's Encrypt для обеспечения защищённого HTTPS-соединения.

Во второй части работы был реализован механизм автоматического получения и обновления токена доступа для сервиса GigaChat с использованием Bash-скрипта, Systemd Service и Systemd Timer. Настроено безопасное хранение секретных данных с ограничением прав доступа. На базе Nginx был реализован обратный прокси-сервер для взаимодействия с внешними LLM-сервисами через единый корпоративный веб-интерфейс.

Дополнительно была разработана веб-страница для работы с языковой моделью GigaChat. Реализовано получение списка доступных моделей через API, отправка пользовательских запросов и потоковый вывод ответа в режиме реального времени. Интерфейс был оформлен с использованием современных HTML, CSS и JavaScript технологий.

В результате выполнения лабораторной работы были освоены принципы настройки веб-серверов, работы с SSL-сертификатами, автоматизации задач средствами Systemd, взаимодействия с REST API, организации обратного проксирования и создания пользовательских веб-интерфейсов для работы с системами искусственного интеллекта.
