## [Install](https://clickhouse.com/docs/ru/getting-started/tutorial)

Чтобы законнектиться снаружи, в **/etc/clickhouse-server/config.xml** раскомментировать строку
```shell
<listen_host>::</listen_host>
```

Данные находятся тут 
```
# cd /var/lib/clickhouse
# du -sh
34M     .
```

> For MC: Press Ctrl + Space on current directory and this will show it size.

> Просто считайте что вся дира **var/lib/clickhouse** это данные и монтируйте ее

![ex](https://github.com/AV-ghub/ClickHouse/blob/main/001%20Install/res/1.jpg)

В каталоге **metadata** хранятся описания .sql таблиц. Там внутри есть UUID этот UUID это имя каталога в store в data симлинки чтобы сохранить совместимость с прошлой структурой, с базами ordinary

> Есть табличка весом в 300гб, необходимо перелить все данные из нее в другую табличку. Насколько хорошая идея делать "INSERT INTO SELECT *"? Клик умеет такие операции оптимизировать?

Такую мелочь за 100 сек проливает
Делаем так в том числе и с табличками выше 1ТБ и из удаленного сервера с remote функцией

[order by](https://clickhouse.com/docs/ru/sql-reference/statements/select/order-by)   
Существует возможность выполнять сортировку во внешней памяти (с созданием временных файлов на диске), если оперативной памяти не хватает. Для этого предназначена настройка **max_bytes_before_external_sort**. Если она выставлена в 0 (по умолчанию), то внешняя сортировка выключена. Если она включена, то при достижении объёмом данных для сортировки указанного количества байт, накопленные данные будут отсортированы и сброшены во временный файл. Файлы записываются в директорию **/var/lib/clickhouse/tmp/** (по умолчанию, может быть изменено с помощью параметра **tmp_path**) в конфиге.

> Когда настройка [optimize_read_in_order](https://clickhouse.com/docs/ru/operations/settings/settings#optimize_read_in_order) отключена, при выполнении запросов SELECT табличные индексы не используются.

## [Перемещение каталога данных](https://stackoverflow.com/a/72852943)
#### Остановка и перемещение
```
$ systemctl stop clickhouse-server
or
$ /etc/init.d/clickhouse-server stop --now
$ systemctl status clickhouse-server
$ sudo cp -a /var/lib/clickhouse /home/data
$ sudo mv /var/lib/clickhouse /var/lib/clickhouse_bak

cd /etc/clickhouse-server
sudo nano config.xml
```
#### Корректировка
```
/var/lib/ -> /home/data/
/var/lib/clickhouse/ -> /home/data/clickhouse/
```
#### Старт сервиса
```
$ systemctl start clickhouse-server --now
$ systemctl status clickhouse-server
```
#### Зачистка
```
sudo rm -Rf /var/lib/clickhouse_bak
```







## [Установка и использование ClickHouse на Linux](https://www.dmosk.ru/miniinstruktions.php?mini=clickhouse-linux)

В данной инструкции мы разберемся с базовыми действиями по администрированию ClickHouse — установкой, настройкой и выполнением некоторых запросов. Мы будем работать на системах Linux — Ubuntu и Rocky Linux.

### Установка и запуск ClickHouse
Перед установкой мы должны убедиться, что процессор на сервере поддерживает SSE версии 4.2. Это делается командой:
```
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```
Мы должны увидеть:
```
SSE 4.2 supported
```
В противном случае, необходимо выполнить сборку. Подробнее процесс описан на сайте разработчика.

Мы же предполагаем, что наш сервер поддерживает нужную инструкцию процессора. Поэтому установка будет выполняться из репозитория.

В родных репозиториях deb и rpm нет ClickHouse. Для начала мы выполним их конфигурирование и уже после — установку базы данных. В зависимости от типа дистрибутива Linux, наши действия будут отличаться.

### Установка на RPM
Для добавления репозитория мы будем использовать yum-utils:
```
yum install yum-utils

yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
```
Теперь устанавливаем кликхаус командой:
```
yum install clickhouse-server clickhouse-client
```
Для установки конкретной версии, если она есть в репозитории, вводим название пакета с указанием самой версии, например:
```
yum install clickhouse-server-23.1.3.5 clickhouse-client-23.1.3.5
```
Запускаем сервис и разрешаем его автозапуск:
```
systemctl enable clickhouse-server --now
```
### Установка на Deb
Устанавливаем пакеты:
```
apt install apt-transport-https ca-certificates dirmngr
```
* где:

apt-transport-https — для возможности взаимодействовать с репозиторими по https.
ca-certificates — набор корневых сертификатов.
dirmngr — для управления сетевыми сертификатами.

Установим gpg-ключ репозитория с сервера ключей:
```
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754
```
Добавим репозиторий:
```
echo "deb https://packages.clickhouse.com/deb stable main" > /etc/apt/sources.list.d/clickhouse.list
```
Обновим кэш:
```
apt update
```
Можно устанавливать clickhouse:
```
apt install clickhouse-server clickhouse-client
```
Запускаем сервис и разрешаем его автозапуск:
```
systemctl enable clickhouse-server --now
```
Управление доступом
После установки clickhouse мы можем подключиться к серверу без ввода логина и пароля под пользователем default. Для этого достаточно в консоли ввести команду:
```
clickhouse-client
```
Для контроля доступа к базе в официальной документации рекомендуется использовать SQL-ориентированное управление. Нам необходимо выполнить следующие шаги:

### Включить SQL-ориентированный контроль доступа. Создать суперпользователя. Удалить доступ для пользователя default.
Рассмотрим эти шаги подробнее.

1) Открываем файл:
```
vi /etc/clickhouse-server/users.xml
```
В разделе users для пользователя default задаем опции access_management, named_collection_control, show_named_collections и show_named_collections_secrets:
```
<clickhouse>
    ...
    <users>
        ...
        <default>
            ...
            <access_management>1</access_management>
            <named_collection_control>1</named_collection_control>
            <show_named_collections>1</show_named_collections>
            <show_named_collections_secrets>1</show_named_collections_secrets>
        </default>
    </users>
    ...
</clickhouse>
```
* желтым показаны опции, которым необходимо задать значение 1.

Для применения настроек перезапускаем сервис:
```
systemctl restart clickhouse-server
```
2) Для создания пользователя подключаемся к серверу:
```
clickhouse-client
```
Создаем учетную запись для суперпользователя:
```
:) CREATE USER root HOST LOCAL IDENTIFIED WITH sha256_password BY 'password';
```
* в данном примере мы создали пользователя root, которому можно подключаться только с локального хоста (HOST LOCAL) и паролем password.

Даем ему полные права на все базы и таблицы:
```
:) GRANT ALL ON *.* TO root WITH GRANT OPTION
```
Если мы получим ошибку с сообщением, что у пользователя default не хватает прав, выполняем команду:
```
GRANT ALL ON *.* TO default WITH GRANT OPTION;
```
Выходим из командной оболочки:
```
:) quit
```
Проверяем доступ, попробовав подключиться к серверу баз данных:
```
clickhouse-client -uroot --ask-password
```
Система запросит пароль. Вводим тот, что задали при создании пользователя (в нашем случае это password).

В итоге, мы должны увидеть приглашение на ввод SQL-команд:
```
:)
```
Теперь можно создать пользователей для подключения к базе. Команда подобна той, что мы использовали для создания пользователя root:
```
:) CREATE USER <имя пользователя> HOST LOCAL IDENTIFIED WITH sha256_password BY '<пароль>';
```
* подробнее процесс описан на официальном сайте.

Задать права можно командой:
```
:) GRANT ALL ON <имя базы>.<имя таблицы> TO <имя пользователя>
```
Список пользователей можно посмотреть командой:
```
:) SELECT name, host_ip, host_names FROM system.users
```
3) В официальной документации рекомендуют ограничить доступ пользователю default, оставив права только на чтение.

Для этого открываем файл:
```
vi /etc/clickhouse-server/users.xml
```
В разделе users для default приводим опции к виду:
```
<clickhouse>
    ...
    <users>
        ...
        <default>
            ...
            <profile>readonly</profile>
            ...
            <access_management>0</access_management>
        </default>
    </users>
    ...
</clickhouse>
```
* в данном примере для пользователя default мы задаем профиль readonly и возвращаем значение для access_management в 0.

Перезапускаем сервис для сохранения настроек:
```
systemctl restart clickhouse-server
```
Подключение по сети
По умолчанию, кликхаус слушает запросы на локальной петле и не принимает запросов по сети. Для решения задачи нам нужно сделать следующее:

Разрешить сервису обработку сетевых запросов.
Создать учетную запись для подключения с удаленного хоста.
Рассмотрим пример настройки сервера и подключения к нему с клиента.

Настройка сервера
Открываем файл:
```
vi /etc/clickhouse-server/config.xml
```
Приводим опцию к виду:
```
<listen_host>0.0.0.0</listen_host>
```
* в данном примере мы говорим серверу слушать на всех сетевых адресах IPv4. Обратите внимание, что данная опция есть в конфигурационном файле, но она закомментирована.

Обратите внимание, что в конфигурационном файле по умолчанию все опции listen_host закомментированы с помощью тегов <!-- ... -->. Их нужно убрать.

Для применения настроек вводим:
```
systemctl restart clickhouse-server
```
Проверить, что сервер стал слушать на всех интерфейсах IPv4 можно с помощью команды:
```
ss -tunlp | grep :9000
```
Теперь заходим в оболочку SQL, например, с помощью ранее созданной учетной записью root:
```
clickhouse-client -uroot --ask-password
```
И создаем учетную запись для подключения по сети:
```
:) CREATE USER remote HOST IP '192.168.100.0/24' IDENTIFIED WITH sha256_password BY 'password';
```
* в нашем примере мы создадим учетную запись remote с паролем password для подключения к нашему серверу из локальной сети 192.168.100.0/24. Мы также можем указать не подсеть, а конкретный узел, с которого пользователю будет разрешено подключение.

Если на сервере используется брандмауэр, то необходимо открыть порт 9000. В зависимости от утилиты управления, наши команды будут отличаться.

а) для Iptables:
```
iptables -I INPUT -p tcp --dport 9000 -j ACCEPT
```
Для сохранения правил используем iptables-persistent:
```
apt install iptables-persistent

netfilter-persistent save
```
б) для firewalld:
```
firewall-cmd --add-port=9000/tcp --permanent
```
И применяем настройку:
```
firewall-cmd --reload
```
Теперь можно попробовать подключиться с другого узла.

### Установка клиента
На сервер необходимо установить клиента. В зависимости от типа дистрибутива Linux, наши действия будут отличаться.

#### а) на RPM.
```
yum install yum-utils

yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
```
Теперь устанавливаем клиента командой:
```
yum install clickhouse-client
```
#### б) на Deb.

Устанавливаем пакеты:
```
apt install apt-transport-https ca-certificates dirmngr
```
Установим gpg-ключ репозитория с сервера ключей:
```
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754
```
Добавим репозиторий:
```
echo "deb https://packages.clickhouse.com/deb stable main" > /etc/apt/sources.list.d/clickhouse.list
```
Обновим кэш:
```
apt update
```
Можно устанавливать клиента:
```
apt install clickhouse-client
```
Подключение к серверу
Теперь можно подключиться клиентом к серверу:
```
clickhouse-client -h192.168.100.15 -uremote --ask-password
```
* где:

192.168.100.15 — адрес сервера, к которому мы подключаемся.
remote — имя учетной записи, под которой мы подключаемся к серверу.
ask-password — клиент должен запросить пароль для пользователя.
После ввода команды система попросить ввести пароль:

Password for user (remote):

В итоге, мы должны увидеть приглашение к вводу команд SQL:
```
:)
```

