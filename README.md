Содержание
================

*   [Общая информация](#общая-информация)
*   [Первичная настройка](#первичная-настройка)
*   [Параметры конфигурации](#параметры-конфигурации)
*   [Настройка PAC](#настройка-pac)

## Общая информация

**smart-proxy** - скрипт на bash запускающий SOCKS прокси и простой HTTP сервер со сгенерированным Proxy Auto-Configuration (PAC).  
Изначально скрипт smart-proxy написан с целью запуска на Linux/Mac, но при необходимости может быть запущен в Windows поверх [WSL](https://learn.microsoft.com/ru-ru/windows/wsl/install) или [CygWin](https://cygwin.com/)  
  
PAC файл автоматически генерируется при запуске или вручную по мере необходимости из файла с хостами `${HOME}/.smart-proxy/hosts_list`
  
PAC содержит домены и IP-адреса ресурсов, а также прокси-сервер, через который нужно точечно перенаправлять трафик по указанным ресурсам.  
  
Идея с PAC не нова, и уже имеется немалоизвестный проект [АнтиЗапрет](https://antizapret.prostovpn.org) чем и вдохновлен smart-proxy  
Отличие от АнтиЗапрета заключается в использовании собственного прокси и пользовательских списков для проксирования.  
Прокси SOCKS запускается поверх SSH туннеля. SSH как туннель **выбран не случайно:**  

*   Протокол SSH пока не блокируется
*   Не требует сложной настройки
*   SSH по умолчанию установлен на всех ОС из коробки

## Первичная настройка

#### Настройки ниже необходимы если доступ к серверу по SSH не настроен

*   Генерируем ssh ключ без пароля:

    ```bash
    $ ssh-keygen -t ed25519 -N "" -C 'smart-proxy' -f ${HOME}/.ssh/ed25519_smart-proxy
    ```

*   Копируем содержимое в **${HOME}/.ssh/ed25519\_smart-proxy.pub** в файл **~/.ssh/authorized\_keys** на удаленном сервере вручную или командой
    ```bash
    $ ssh-copy-id -i ${HOME}/.ssh/ed25519_smart-proxy.pub ИМЯ_ПОЛЬЗОВАТЕЛЯ_ВАШЕГО_СЕРВЕРА@IP_АДРЕС_ВАШЕГО_СЕРВЕРА
    ```
*   В **${HOME}/.ssh/config** добавляем конфиг:
    ```
    Host smart-proxy
        HostName IP_АДРЕС_ВАШЕГО_СЕРВЕРА
        User ИМЯ_ПОЛЬЗОВАТЕЛЯ_ВАШЕГО_СЕРВЕРА
        IdentitiesOnly yes # используем авторизацию исключительно по ключу
        IdentityFile ~/.ssh/ed25519_smart-proxy.pub
    ```
    

*   Пытаемся подключиться по SSH:
    ```bash
    $ ssh smart-proxy
    ```

Если всё в порядке, идём дальше; в противном случае разбираемся с SSH.
Важно! Каждый раз при запуске скрипта нужно будет вводить пароль, если он установлен для ключа.
В качестве выхода из ситуации можно использовать SSH-агент.
Также можно использовать альтернативный файл настроек или другое имя ssh подключения.
Подробности в [Параметры конфигурации](#параметры-конфигурации)

#### Первый запуск
*   Склонируем репозиторий
    ```bash
    $ git clone https://github.com/mukhumaev/smart-proxy && cd smart-proxy
    ```
*   Инициализируем
    ```bash
    $ ./smart-proxy init
    ```
Если у вас Linux, то после инициализации скрипт автоматически установится в `${HOME}/.local/bin/smart-proxy`, в противном случае необходимо самостоятельно установить скрипт в одну из директорий переменной `${PATH}` для запуска без указания полного пути
*   Добавляем домены и IP адреса для проксирования согласно описанию `Настройка списка доменов и IP адресов для просирования` ниже
*   Запускаем скрипт
    ```bash
    $ /path/to/smart-proxy start
    ```
*   Настраиваем PAC согласно инструкции [Настройка PAC](#настройка-pac)
*   Радуемся жизни

#### Настройка списка доменов и IP адресов для просирования

Домены и IP адреса для проксирования указываются в файле `${HOME}/.smart-proxy/hosts_list`.  
Домены можно указывать с \* тогда будут учтены все поддомены родительского домена. Пример `hosts_list` для проксирования YouTube:  
```
*.googlevideo.com
*.youtube.com
*.youtu.be
*.ytimg.com
*.ggpht.com
*.youtubei.googleapis.com
```

IP адреса настраиваются аналогично, только вместо \* указывается подсеть. Пример для проксирования 2ip.ru по IP и подсети `172.19.12.16/29`:
```
195.201.201.32  # IP сайта 2ip.ru
172.19.12.16/29 # подсеть
```
    

При изменении файла `${HOME}/.smart-proxy/hosts_list` необходимо перезапускать smart-proxy ИЛИ пререгенерировать файлы:
```
$ /path/to/smart-proxy restart # команда перезапуска
$ /path/to/smart-proxy regen-files # перегенерировать файлы
```

## Параметры конфигурации

Файл конфигурации по умолчанию расположен в `${HOME}/.smart-proxy/config` и позволяет задавать следующие переменные:

|Переменная|Значение по умолчанию|Комментарий|
|-|-|-|
|**PROXY\_IP**|127.0.0.1|IP адрес для запуска SSH прокси|
|**PROXY\_PORT**|61942|Порт для SSH прокси|
|**PROXY\_AUTORESTART**|true|Рестарт SSH прокси при аварийном завершении|
|**FILES\_GEN\_AUTORESTART**|false|Автоматически перегенерировать файлы для HTTP сервера при изменении файла с доменами или IP адресами|
|**FILES\_GEN\_INTERVAL**|10|Интервал проверки файла на наличие изменений в секундах. Не используется при FILES\_GEN\_AUTORESTART=false|
|**IP\_CHECKER\_HOST**|ifconfig.me|Внешний хост для проверки работы прокси. При обращении через curl должен возвращать IP адрес. Пример таких сервисов: 2ip.ru, eth0.me, icanhazip.com, api64.ipify.org, ipinfo.io/ip|
|**HTTP\_SERVER\_IP**|127.0.0.1|IP адрес HTTP сервера|
|**HTTP\_SERVER\_PORT**|8080|Порт для HTTP сервера|
|**HTTP\_AUTORESTART**|true|Рестарт HTTP сервера при аварийном завершении|
|**SSH\_CONFIG**|true|Использовать SSH конфиг файл SSH\_CONFIG\_FILE для прокси|
|**SSH\_CONFIG\_FILE**|${HOME}/.ssh/config|Путь к SSH конфигу|
|**SSH\_HOST**|smart-proxy|Хост для организации SSH подключения. Если SSH\_CONFIG=true то хост из конфига указанного в переменной SSH\_CONFIG\_FILE, в противном случае - хост для прямого подключения|
|**SSH\_PORT**|22|Порт SSH удаленного хоста. Не используется при SSH\_CONFIG=true|
|**SSH\_IDENTITY**|${HOME}/.ssh/id\_rsa|SSH ключ для подключения к удаленному хосту. Не используется при SSH\_CONFIG=true|


## Настройка PAC

Ниже описаны инструкции для настройки PAC в браузерах и ОС.  
Важно! Настройки, адреса и данные для подключения действительны для конфигурации по умолчанию. Значения могут отличаться, если переопределены переменные, указанные в [Параметры конфигурации](#параметры-конфигурации)

### Настройка на уровне браузера

#### Для Chrome, Яндекс, Vivaldi, Opera, Edge:

1.  Установить расширение [FoxyProxy](https://chromewebstore.google.com/search/FoxyProxy)
2.  Открыть настройки расширения **FoxyProxy**
3.  Перейти в: **Настройки > Импортировать настройки > Import from URL**
4.  Вставить ссылку [http://127.0.0.1:8080/foxy_proxy.json](http://127.0.0.1:8080/foxy_proxy.json)
5.  Нажмите кнопку **Импортировать настройки**, затем **Сохранить**
6.  Нажмите на иконку **FoxyProxy** в панели задач браузера и выберите **smart-proxy**

#### Для Firefox:

1.  Перейдите в: **Меню > Настройки > Основные > Настройки сети > Настроить**.
2.  Вставьте ссылку [http://127.0.0.1:8080/proxy.pac](http://127.0.0.1:8080/proxy.pac) в поле "**URL автоматической настройки прокси**" ([подробнее](https://support.mozilla.org/ru/kb/parametry-soedineniya-v-firefox))
3.  Или воспользуйтесь расширением [FoxyProxy](https://addons.mozilla.org/ru/firefox/addon/foxyproxy-standard/) и настройте по инструкции выше.

### Настройка на уровне ОС

#### Linux (Gnome):

1.  Откройте **Настройки > Сеть > Прокси**
2.  Активируйте **Сетевой прокси**
3.  В выпадающем списке **Конфигурация** выберите **Автоматически**
4.  В поле **URL конфигурации** вставьте [http://127.0.0.1:8080/proxy.pac](http://127.0.0.1:8080/proxy.pac)

Аналогичная настройка через терминал:
```
$ gsettings set org.gnome.system.proxy mode 'auto'
$ gsettings set org.gnome.system.proxy autoconfig-url 'http://127.0.0.1:8080/proxy.pac'
```
#### macOS:

1.  В меню **Apple > Системные настройки** выберите **Сеть**.
2.  Справа выберите сетевую службу, затем нажмите **Подробнее**
3.  Перейдите на вкладку **Прокси**
4.  Включите **Автоконфигурацию прокси**, затем введите адрес [http://127.0.0.1:8080/proxy.pac](http://127.0.0.1:8080/proxy.pac).
5.  Нажмите **ОК**

#### Windows 10:

1.  Откройте настройки: **Пуск > Параметры > Сеть и Интернет > Proxy**.
2.  Включите **Использовать сценарий установки**
3.  В поле **Адрес скрипта** введите адрес [http://127.0.0.1:8080/proxy.pac](http://127.0.0.1:8080/proxy.pac) и нажмите **Сохранить**

#### Windows 11:

1.  Откройте настройки: **Пуск > Параметры > Сеть и Интернет > Proxy**.
2.  Выберите **Настроить** рядом с пунктом **Использовать сценарий установки**.
3.  В диалоговом окне **Изменение скрипта установки** включите опцию, введите адрес [http://127.0.0.1:8080/proxy.pac](http://127.0.0.1:8080/proxy.pac) и нажмите **Сохранить**.
