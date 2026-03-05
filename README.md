# Ручная установка Pterodactyl Panel и Wings Daemon
> Я делаю эту инструкцию по-настоящему универсальной – чтобы она одинаково хорошо работала и на домашнем ПК за NAT, и на облачном VPS, с доменом или без.  
>
> Скоро здесь появятся подробные гайды:
> - **Проброс портов** – открываем серверы для друзей и игроков из любой точки мира / размещаем панель в интернете
> - **SSL (HTTPS)** – бесплатные сертификаты Let's Encrypt для панели и wings
> - **Первый сервер** – запустим свой первый игровой сервер
> - **Домен и не только** – как привязать свой домен и настроить DNS
>
> Следите за обновлениями – скоро станет ещё проще развернуть идеальный игровой хостинг!

А сейчас данное руководство предназначено для развёртывания панели управления игровыми серверами Pterodactyl и её демона Wings на чистом сервере с Ubuntu 24.04 в локальной сети. Вся конфигурация выполняется без использования SSL (только HTTP), что идеально подходит для внутреннего использования, тестирования или начального знакомства с системой.

## Что будет установлено
- **Веб-панель Pterodactyl** – основной интерфейс управления (PHP 8.3 + Nginx + MySQL + Redis)
- **Демон Wings** – компонент, запускающий игровые серверы в изолированных Docker-контейнерах
- Все необходимые зависимости и инструменты (curl, git, Docker, Composer и др.)

## Рекомендуемые аппаратные требования
- **Процессор:** от 2 ядер (для активной работы желательно 4 и более)
- **Оперативная память:** минимум 2 ГБ для панели + дополнительно на каждую запущенную игровую ноду (обычно от 1 ГБ на ноду)
- **Дисковое пространство:** не менее 20 ГБ для системы и панели; фактический объём зависит от количества и типа игровых серверов.

## Программное обеспечение (версии, используемые в инструкции)
- **Операционная система:** Ubuntu 22.04 / 24.04 (чистая установка)
- **Веб-сервер:** Nginx (в роли reverse proxy)
- **PHP:** 8.3 с расширениями (cli, common, gd, mysql, mbstring, bcmath, xml, curl, zip, fpm, intl)
- **База данных:** MySQL 8.0+ или MariaDB 10.6+
- **Кеш и очереди:** Redis 6+
- **Контейнеризация:** Docker Engine (с плагином Docker Compose)
- **Менеджер PHP-пакетов:** Composer 2

В процессе установки все перечисленные компоненты будут настроены на автоматический запуск. Необходимые порты будут открыты в брандмауэре UFW по мере необходимости — точные номера портов указаны в соответствующих разделах инструкции.

## Оглавление

- [Подготовка системы / зависимостей](#zavisimosti)
  - [Обновление системы и установка базовых пакетов](#mainupdate)
  - [PHP 8.3](#php)
  - [MySQL 8+](#mysql)
  - [Redis Server](#redis)
  - [Nginx](#nginx)
  - [Docker](#docker)
  - [Composer](#composer)
  - [Очередное обновление системы](#opyatupdate)
  - [Включение автозапуска сервисов](#vklautozapysk)
  - [Настройка брандмауэра (UFW)](#ufwbaseallow)
  - [Проверка итоговой конфигурации](#testconfigbase)
- [Установка Pterodactyl Panel](#installpanel)
  - [Подготовка базы данных](#mysqlpogifg)
  - [Загрузка и установка файлов панели](#uploadinstallfilespanel)
  - [Настройка прав доступа](#changechmodpanel)
  - [Установка зависимостей Composer](#insatllcomposerzavisimost)
  - [Настройка окружения](#nastroikaenv)
  - [Заполнение базы данных](#zapolneniyedbmysql)
  - [Настройка планировщика задач](#crontabblabal)
  - [Настройка демона](#daemonnastroika)
  - [Настройка Nginx](#nginxwebserver)
  - [Заходим на сайт](#zahodimvpanelwww)
- [Установка Wings](#installwings)
  - [Подготовка системы](#podgotovkawingssystem)
  - [Создание конфигурации logrotate](#logrotatelocation)
  - [Загрузка бинарного файла wings](#zagryzkawingsfiile)
  - [Создание systemd-сервиса](#systemdserviswings)
  - [Настройка конфигурации](#configurationconfigyml)
  - [Установка необходимых прав](#chownchmodvladelec640)
  - [Запуск и включение автозапуска](#zapyskiavutozapysk)
  - [Настройка брандмауэра (UFW)](#ufwopyatnastoika)
- [Финал](#final)

<a name="zavisimosti"></a>
## Подготовка системы / зависимостей
<a name="mainupdate"></a>
### Обновление системы и установка базовых пакетов
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl tar unzip git software-properties-common apt-transport-https ca-certificates
```
<a name="php"></a>
### PHP 8.3 и расширения
Добавляем PPA с актуальными версиями PHP и устанавливаем PHP-FPM вместе с основными расширениями
```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install -y php8.3 php8.3-cli php8.3-common php8.3-gd php8.3-mysql \
php8.3-mbstring php8.3-bcmath php8.3-xml php8.3-curl php8.3-zip \
php8.3-fpm php8.3-intl
```
Проверьте версию и статус сервиса
```bash
php -v
sudo systemctl status php8.3-fpm
```
<a name="mysql"></a>
### MySQL Server
Установите MySQL и выполните скрипт начальной настройки безопасности
```bash
sudo apt install -y mysql-server
sudo mysql_secure_installation
```
- хотите ли вы актвировать проверку пароля: yes
- 0-2 (какая сложность пароля должна быть): 0
- удалить анонимного пользователя: yes
- Запретить удаленное root подключение: yes
- Удалить тестовую базу данных: yes
- Перезагрузить таблицу привилегий: yes
  
Проверьте версию и статус сервиса
```bash
mysql --version
sudo systemctl status mysql
```
<a name="redis"></a>
### Redis
Установите Redis и проверьте статус сервиса
```bash
sudo apt install -y redis-server
sudo systemctl status redis-server
```
<a name="nginx"></a>
### Nginx
Установите и запустите веб сервер Nginx
```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```
Проверьте доступность локально
```bash
curl -I http://localhost
```
<a name="docker"></a>
### Docker (engine / compose)
Добавьте официальный репозиторий Docker и установите компоненты
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
Проверьте версию и статус сервиса
```bash
docker --version
sudo systemctl status docker
```
<a name="composer"></a>
### Composer
Установите менеджер зависимостей PHP и проверьте версию 
```bash
sudo apt install -y composer
composer --version
```
<a name="opyatupdate"></a>
### Очередное обновление системы
```bash
sudo apt update && sudo apt upgrade -y
```
<a name="vklautozapysk"></a>
### Включение автозапуска сервисов
Настройте автоматический запуск всех необходимых служб при загрузке системы
```bash
sudo systemctl enable php8.3-fpm mysql nginx redis-server docker
sudo systemctl start php8.3-fpm mysql nginx redis-server docker
```
<a name="ufwbaseallow"></a>
### Настройка брандмауэра (UFW)
Установите политики по умолчанию (исходящие соединения: разрешить все | входящие: запретить все, что не разрешены)
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Разрешите SSH и Nginx Full, затем включите фаервол
```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```
Проверьте работает ли фаервол
```bash
sudo systemctl status ufw
```
<a name="testconfigbase"></a>
### Проверка итоговой конфигурации
Убедитесь, что все сервисы работают
```bash
sudo systemctl status php8.3-fpm mysql nginx redis-server docker ufw
```

<a name="installpanel"></a>
## Установка Pterodactyl Panel
<a name="mysqlpogifg"></a>
### Подготовка базы данных
Зайдите в MySQL и создайте базу данных и пользователя для панели
```bash
sudo mysql
```
Выполните следующие команды в консоли MySQL. Замените `<password>` на надёжный пароль
```bash
CREATE DATABASE panel;
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit
```
<a name="uploadinstallfilespanel"></a>
### Загрузка и установка файлов панели
Создайте директорию и скачайте последнюю версию панели
```bash
sudo mkdir -p /var/www/pterodactyl
cd /var/www/pterodactyl
sudo curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
sudo tar -xzvf panel.tar.gz
```
<a name="changechmodpanel"></a>
### Настройка прав доступа
Установите правильные владельца и разрешения
```bash
sudo chmod -R 755 storage/* bootstrap/cache/
sudo chown -R www-data:www-data /var/www/pterodactyl
```
<a name="insatllcomposerzavisimost"></a>
### Установка зависимостей Composer
От имени пользователя `www-data` выполните установку PHP-пакетов
```bash
sudo -u www-data composer install --no-dev --optimize-autoloader
```
Появится крансая ошибка `No application encryption key has been specified.` - не переживайте, это нормально
<a name="nastroikaenv"></a>
### Настройка окружения
Скопируйте файл-пример конфигурации и сгенерируйте ключ приложения
```bash
sudo -u www-data cp /var/www/pterodactyl/.env.example /var/www/pterodactyl/.env
sudo -u www-data php artisan key:generate --force
```
Теперь отредактируйте файл `.env`, указав параметры подключения к базе данных и Redis
```bash
sudo nano /var/www/pterodactyl/.env
```
Найдите и измените следующие строки (подставьте свои данные)
```bash
APP_TIMEZONE=UTC
APP_URL=http://panel.example.com
DB_PASSWORD=
```
Пример (узнайте ip машины с помощью ip a)
```bash
APP_TIMEZONE=Europe/Moscow
APP_URL=http://192.168.77.12
DB_PASSWORD=qweasdzxc
```
Проверьте начилие строк с Redis, в случае отсутствия - добавьте их вручную (ничего не меняя)
```bash
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```
<a name="zapolneniyedbmysql"></a>
### Заполнение базы данных
Выполните миграцию базы данных
```bash
sudo -u www-data php artisan migrate --force
```
Запустите скрипт для создания первого пользователя
```bash
sudo -u www-data php artisan p:user:make
```
- Пользователь будет администратором: yes
- email: example@mail.com
- username: admin
- firstname: admin
- lastname: admin
- password: ochenslojniyparol 
<a name="crontabblabal"></a>
### Настройка планировщика задач
Измените часовой пояс для корректной работы (Europe/Moscow замените на ваш)
```bash
sudo timedatectl set-timezone Europe/Moscow
```
Откройте crontab для пользователя www-data
```bash
sudo crontab -u www-data -e
```
При первом запуске выберите редактор (например: nano)
Добавьте в файл
```bash
* * * * * php /var/www/pterodactyl/artisan schedule:run >> /dev/null 2>&1
```
<a name="daemonnastroika"></a>
### Настройка демона
Создайте systemd файл
```bash
sudo nano /etc/systemd/system/pteroq.service
```
```bash
[Unit]
Description=Pterodactyl Queue Worker
After=redis-server.service

[Service]
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/pterodactyl/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
Затем запустите сервис и проверьте его
```bash
sudo systemctl enable --now pteroq.service
sudo systemctl status pteroq.service
```
<a name="nginxwebserver"></a>
### Настройка Nginx
Удалите стандартный конфиг и создайте для Pterodactyl
```bash
sudo rm /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-available/pterodactyl
```
Вставляйте в него
```bash
server {
    listen 80;
    server_name 127.0.0.1;

    root /var/www/pterodactyl/public;
    index index.php;

    access_log /var/log/nginx/pterodactyl.app-access.log;
    error_log  /var/log/nginx/pterodactyl.app-error.log error;

    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffering off;
        fastcgi_read_timeout 120s;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
Активируйте конфигурацию, проверьте её и перезапустите nginx (если ошибок нет)
```bash
sudo ln -s /etc/nginx/sites-available/pterodactyl /etc/nginx/sites-enabled/pterodactyl
sudo nginx -t
sudo systemctl restart nginx
```
<a name="zahodimvpanelwww"></a>
### Заходим на сайт
Панель доступна по адресу:
- Локально на сервере: http://127.0.0.1
- В локальной сети: http://192.168.xx.xx (можете узнать ip с помощью ip a)

<a name="installwings"></a>
## Установка Wings
Wings — это высокопроизводительный демон для Pterodactyl, который запускает и управляет игровыми серверами в изолированных Docker-контейнерах. Устанавливается на той же машине, где работает панель, либо на отдельной ноде.
<a name="podgotovkawingssystem"></a>
### Подготовка системы
Создайте отдельного системного пользователя для wings и необходимые директории
```bash
sudo useradd -r -s /bin/false pterodactyl
sudo mkdir -p /etc/pterodactyl /var/lib/pterodactyl /var/log/pterodactyl /run/wings
sudo chown -R pterodactyl:pterodactyl /var/lib/pterodactyl /var/log/pterodactyl /run/wings
sudo chmod 755 /var/log/pterodactyl /run/wings
```
Даём пользователю доступ к Docker
```bash
sudo usermod -aG docker pterodactyl
```
<a name="logrotatelocation"></a>
### Создание конфигурации logrotate
```bash
sudo nano /etc/logrotate.d/wings
```
Вставляйте содержимое
```bash
/var/log/pterodactyl/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 pterodactyl pterodactyl
    sharedscripts
    postrotate
        systemctl restart wings > /dev/null 2>&1 || true
    endscript
}
```
<a name="zagryzkawingsfiile"></a>
### Загрузка бинарного файла wings
Скачайте последнюю версию wings в `/usr/local/bin` и сделайте её исполняемой
```bash
cd /usr/local/bin
sudo curl -L -o wings https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_amd64
sudo chmod +x wings
```
Проверим установились ли крылья
```bash
wings -h
```
<a name="systemdserviswings"></a>
### Создание systemd-сервиса
Чтобы wings автоматически запускался при загрузке системы, создайте unit-файл
```bash
sudo nano /etc/systemd/system/wings.service
```
Вставьте следующее содержимое
```bash
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service

[Service]
User=pterodactyl
Group=pterodactyl
WorkingDirectory=/etc/pterodactyl
LimitNOFILE=4096
ExecStart=/usr/local/bin/wings
ExecReload=/bin/kill -SIGTERM $MAINPID
Restart=on-failure
RestartSec=5s
ExecStartPre=+/usr/bin/mkdir -p /run/wings
ExecStartPre=+/usr/bin/chown pterodactyl:pterodactyl /run/wings
ExecStartPre=+/usr/bin/chmod 755 /run/wings

[Install]
WantedBy=multi-user.target
```
<a name="configurationconfigyml"></a>
### Настройка конфигурации
Конфигурационный файл `/etc/pterodactyl/config.yml` можно создать автоматически через панель управления. Для этого:
- Войдите в панель Pterodactyl как администратор.
- Перейдите в **Администрирование → Locations**
- Нажмите **Create New**
- Заполните поля
  - Short Code: короткий код (например, msk или nyc)
  - Description: описание (например, Moscow Data Center)
- Нажмите **Create**
- Перейдите в **Администрирование → Nodes → Create New**.
- Заполните данные узла:
    - Укажите название узла
    - Выберите созданную локацию
    - В поле FQDN введите локальный IP или домен вашего сервера
    - Use SSL Connection замените на Use HTTP Connection
    - Остальные настройки (память, диск, порты) настройте под ваши хотелки
- После сохранения узла откройте его и перейдите на вкладку **Configuration**.
- Скопируйте сгенерированную команду для автоматической настройки
- Выполните эту команду на сервере, где устанавливается wings
Эта команда создаст корректный `/etc/pterodactyl/config.yml` с данными для подключения к панели
При успешном запуске вы увидите сообщение об успешном подключении к панели
<a name="chownchmodvladelec640"></a>
### Установите необходимые права
```bash
sudo chown pterodactyl:pterodactyl /etc/pterodactyl/config.yml
sudo chmod 640 /etc/pterodactyl/config.yml
```
<a name="zapyskiavutozapysk"></a>
### Запуск и включение автозапуска
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wings
```
Проверьте статус
```bash
sudo systemctl status wings
```
<a name="ufwopyatnastoika"></a>
### Настройка брандмауэра (UFW)
Разрешите соединения для портов 8080 (wings) и 2022 (sftp)
```bash
sudo ufw allow 8080
sudo ufw allow 2022
```
Перезапустите и проверьте работает ли фаервол
```bash
sudo systemctl restart ufw
sudo systemctl status ufw
```
Вернитесь в веб-панель, откройте страницу узла. Если wings запущен и настроен правильно, статус узла изменится на зелёный (активный).

<a name="final"></a>
## 🎉 Установка завершена!

Поздравляю! Вы успешно развернули Pterodactyl Panel и Daemon Wings на своём сервере. Теперь вы можете создавать игровые серверы, управлять пользователями и наслаждаться мощью этой платформы.

Если инструкция оказалась полезной, буду благодарен за:
- ⭐ **Звезду** на GitHub – это motivates продолжать развитие.
- 📝 **Сообщения об ошибках** или предложения по улучшению – создавайте issue или пишите напрямую.
- 🔄 **Дополнения** – если вы добавили что-то новое (например, поддержку SSL или проброс портов), делитесь pull request'ами.

Вместе мы сделаем эту инструкцию ещё лучше и доступнее для всех!

**Полезные ссылки:**
- [Официальная документация Pterodactyl](https://pterodactyl.io/)
- [Сообщество Pterodactyl в Discord](https://discord.gg/pterodactyl)

Удачного хостинга! 🚀
