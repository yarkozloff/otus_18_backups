# Настройка бэкапов
## Описание ДЗ
Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client.
Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:
- директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;
- репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;
- имя бекапа должно содержать информацию о времени снятия бекапа;
- глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
- резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
- написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;
- настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.

Запустите стенд на 30 минут.
Убедитесь что резервные копии снимаются.
Остановите бекап, удалите (или переместите) директорию /etc и восстановите ее из бекапа.
Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления.
Формат сдачи ДЗ - vagrant + ansible

## Настройка
### Среда окружения:
```
root@yarkozloff:/otus/backups# hostnamectl | grep "Operating System"
  Operating System: Ubuntu 20.04.3 LTS
root@yarkozloff:/otus/backups# vboxmanage --version
6.1.26_Ubuntur145957
root@yarkozloff:/otus/backups# vagrant --version
Vagrant 2.2.19
```
Также для обоих машин был взят локальный бокс centos7 (при необходимости загрузки облака vagrant просто изменить название на centos/7).
### Настройка сервера backup
На сервере бэкапов устанавливаем необходимые пакеты: epel-release, borgbackup. Также создаем учетку borg и делаем её владельцем для каталога /var/backup, который смонтирован с диска /dev/sdb. Диск необходимо заранее создать в вагрантфайле. Каталог /var/backup зачищаем от лишних файлов. Далее нам понадобятся ssh ключи, которые будут использоваться на клиенте, их переносим в удобное место.
### Настройка сервера client
На клиенте устанавливаем пакеты: epel-release, borgbackup. Генерируем ssh ключи, и переносим ключи с сервера с бэкапа на клиент.
## Тестирование
### Вручную
На клиенте пробуем удаленно инициализировать репозиторий и создать бэкап каталога /etc:
``[root@client ~]# borg init --encryption=repokey borg@192.168.11.160:/var/backup/
Enter new passphrase:
Enter same passphrase again:
Do you want your passphrase to be displayed for verification? [yN]: N

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.11.160/var/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).`

```
Выпонление ручного бэкапа:
```
[root@client ~]# /bin/borg create --stats --list borg@192.168.11.160:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc
Enter passphrase for key ssh://borg@192.168.11.160/var/backup:
...
...
------------------------------------------------------------------------------
Archive name: etc-2022-08-03_20:31:37
Archive fingerprint: a3014a53ec1542488447715aa69cd762db9beff23f75f3557d17e0157fec83b2
Time (start): Wed, 2022-08-03 20:32:00
Time (end):   Wed, 2022-08-03 20:32:06
Duration: 5.96 seconds
Number of files: 1706
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:               28.44 MB             13.50 MB             11.85 MB
All archives:               28.44 MB             13.50 MB             11.85 MB

                       Unique chunks         Total chunks
Chunk index:                    1284                 1698
------------------------------------------------------------------------------
```
Успех.

### Автоматизация
На клиенте бэкапом будет заниматься сервис borg-backup.service, помогать в этом ему будет каждые 5 мин таймер borg-backup.timer. 
