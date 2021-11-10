# Cервера для радиовещания (Centos7 + Icecast)

____
## Оглавление

1. [Установка](#Установка)
2. [Настройка](#Настройка)
3. [Запуск](#Запуск)
4. [Настройка редиректа радиостанции](#Настройка редиректа радиостанции)
5. [Свои плейлисты (Ices)](#Свои плейлисты (Ices)) 
____


## Установка

Устанавливаем расширенный репозиторий epel:
```bash
yum install epel-release
```
Устанавливаем icecast:
```bash
yum install icecast
```
____

## Настройка

Конфигурационный файл для Icecast находится в /etc
```bash
nano /etc/icecast.xml
```
<!-- ... --> является комментарием и не учитывается программой.

### Firewall
Добавить разрешение для 8000 порта в настройки iptables
```bash
iptables -I INPUT 1 -p tcp --dport 8000 -j ACCEPT
```
или настроить Firewalld 
```bash
firewall-cmd --permanent --add-port=8000/tcp
firewall-cmd --reload
```
### отключить SELinux
Посмотреть текущий статус системы безопасности можно командой:
```bash
getenforce
```
Варианты ответа: Enforcing / Permissive / Disabled — включен / только предупреждения / отключен.

#### Разово без перезагрузки
Введите команду:
```bash
setenforce 0
```
После перезагрузки системы все вернется как было. 
#### Отключить навсегда
Чтобы SELinux не запускался после перезагрузки, открываем на редактирование следующий файл:
```bash
nano /etc/selinux/config
```
И редактируем следующую строку:
```bash
...
SELINUX=disabled
...
```
_* disabled отключает selinux, enforcing — включает. (italic)_

### Настройка icecast.xml
```
<clients>100</clients>
Количество одновременно подключенных пользователей. Выставляем желаемое.
<sources>3</sources>
Количество обрабатываемых сервером аудиопотоков. Если надо организовать несколько разных потоков аудио.

<source-password>mypass</source-password>
Пароль для присоединения потока к аудиосерверу IceCast.
< relay-password >mypass< /relay-password >
Пароль, используемый для пересылки аудиопотоков между локальным IceCast-сервером и другим IceCast-сервером. Эти пароли на обоих серверах должны совпадать.

<admin-user>admin</admin-user>
Логин администратора, обслуживающего сервер. По умолчанию admin.
<admin-password >pass</admin-password >
Пароль администратора. Используется для всех административных функций.

<port>8000</port>
Настройка номера TCP-порта. Значение по умолчанию 8000.

<bind-address>192.168.1.11</bind-address>
Привязка к сетевому адресу. Если параметр не указан, используется значение hostname.

```

## Запуск
Разрешаем сервис и запускаем его следующими командами
```bash
systemctl enable icecast

systemctl start icecast
```
Открываем браузер и переходим по пути http://192.168.0.15:8000/

_* где 192.168.0.15 — IP-адрес нашего сервера, который мы прописали в bind-address конфига.(italic)_

## Настройка редиректа радиостанции
Добавляем в icecast.xml

```bash
<relay>
    <server>shoutcast.aichyna.com</server>
    <port>9000</port>
    <mount>/aplus_128</mount>
    <local-mount>/aplus</local-mount>
    <on-demand>0</on-demand>
</relay>
```
* перенаправлений может быть несколько.
* server — имя сервера, с которого берется поток; 
* port — сетевой порт, на котором удаленный сервер отдает поток; 
* mount — точка мониторования на стороне удаленного сервера, с которого берем поток; 
* local-mount — точка монтирования, которая будет использоваться нашим сервером для обращения к настраиваемому потоку; 
* on-demand — если стоит 0, сервер всегда берет поток и проигрывает его, если 1 — только при наличие активных обращений.

перезагружаем IceCast

```bash
systemctl restart icecast
```

## Свои плейлисты (Ices)
Создать свой список музыкальных композиций и передать его серверу Icecast можно с помощью Ices. Для начала, выполним его установку.

#### Установка клиента
Скачать с страницы https://icecast.org/ices/   Ices0 (ices2 не умеет работать с mp3)
```bash
wget http://downloads.us.xiph.org/releases/ices/ices-0.4.tar.gz
```
Распаковываем архив и заходим в каталог:
```bash
tar -zxvf ices*
cd ices*
```
Устанавливаем пакеты, нужные для сборки:
```bash
yum install gcc libxml2-devel libshout-devel gcc-c++
```
Запускаем конфигурирование, сборку и установку:
```bash
./configure
make
make install
```

#### Настройка Ices и запуск плейлиста
Создаем каталог конфигурационного файла и сам файл:
```bash
mkdir /etc/ices
nano /etc/ices/ices.xml
```
```bash
<?xml version="1.0"?>
<ices:Configuration xmlns:ices="http://www.icecast.org/projects/ices">
<Playlist>
  <File>/etc/ices/playlist.rock.txt</File>
  <Randomize>1</Randomize>
  <Type>builtin</Type>
  <Module>ices</Module>
</Playlist>
<Execution>
  <Background>1</Background>
  <Verbose>0</Verbose>
  <BaseDirectory>/etc/ices</BaseDirectory>
</Execution>

<Stream>
  <Server>
        <Hostname>192.168.0.15</Hostname>
        <Port>8000</Port>
        <Password>newpassword</Password>
        <Protocol>http</Protocol>
  </Server>

  <Mountpoint>/rock</Mountpoint>
  <Dumpfile>ices.dump</Dumpfile>
  <Name>Default stream</Name>
  <Genre>Default genre</Genre>
  <Description>Default description</Description>
  <URL>http://192.168.0.15:8000</URL>
  <Public>0</Public>
  <Bitrate>128</Bitrate>
  <Reencode>0</Reencode>
  <Channels>2</Channels>
</Stream>
</ices:Configuration>
```

* где, как правило, редактируется следующее:
* File — путь до файла со списком аудиофайлов.
* Randomize — воспроизведение в случайном порядке.
* Verbose — отладка. Следует поменять на 1, если программа работает не корректно.
* BaseDirectory — рабочий каталог программы. В нем будут храниться pid и log файлы.
* Hostname — адрес нашего сервера icecast.
* Port — порт, на котором слушает сервер icecast.
* Password — пароль для ресурса, который был выставлен в конфигурационном файле icecast.
* Mountpoint — точка монтирования на сервере для плейлиста.
* URL — путь URL до плейлиста.




Создадим список аудиофайлов:
```bash
ls -d $( pwd )/music/rock/ > /etc/ices/playlist.rock.txt
```
* данной командой мы прочитаем содержимое каталога /music/rock и сделаем из его содержимого плейлист для ices.
* по сути, файл playlist.rock.txt должен включать перечень всех аудиофайлов с полным путем до них. Каждый файл с новой строчки.

Запускаем ices:
```bash
ices -c /etc/ices/ices.xml
```
* где /etc/ices/ices.xml — путь до конфигурационного файла.
