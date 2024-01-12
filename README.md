#### SELinux: проблема с удаленным обновлением зоны DNS

Инженер настроил следующую схему:

- ns01 - DNS-сервер (192.168.50.10);
- client - клиентская рабочая станция (192.168.50.15).

При попытке удаленно (с рабочей станции) внести изменения в зону ddns.lab происходит следующее:
```bash
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
>
```
Инженер перепроверил содержимое конфигурационных файлов и, убедившись, что с ними всё в порядке, предположил, что данная ошибка связана с SELinux.

В данной работе предлагается разобраться с возникшей ситуацией.


#### Задание

- Выяснить причину неработоспособности механизма обновления зоны.
- Предложить решение (или решения) для данной проблемы.
- Выбрать одно из решений для реализации, предварительно обосновав выбор.
- Реализовать выбранное решение и продемонстрировать его работоспособность.


#### Формат

- README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них.
- Исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.

### Поиск причин неработоспособности механизма обновления зоны.

Проверил логи SELinux, что бы понять в чем может быть проблема. Для чего использовалал утилиту audit2why  
    cat /var/log/audit/audit.log | audit2why  
На клиенте проблем не обноружил.

Выполнил проверку на сервере ns01.
В логах видно, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.
<image src="./screens/audit2why.jpg" alt="audit2why">

Тут мы также видим, что контекст безопасности неправильный.  
Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге.  
Посмотрим, в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux.  
Для этого воспользуемся командой: semanage fcontext -l  
<image src="./screens/semanage.jpg" alt="semanage">
Изменим тип контекста безопасности для каталога /etc/named: sudo chcon -R -t named_zone_t /etc/named  
После изменения типа контекста безопасности пробуем внести изменения с клиента:
<image src="./screens/add_setting_dns.jpg" alt="add_setting_dns">
Видим, что изменения внесены. Перезагрузим виртуальные машины, что бы убедиться в сохранении настроек.
<image src="./screens/dig.jpg" alt="dig">



