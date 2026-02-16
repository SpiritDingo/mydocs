Ошибка возникает из-за того, что в системе не установлен bc (калькулятор командной строки). В строке 53 скрипта используется bc для вычисления количества дней:

```bash
echo "($end_date_seconds-$now_seconds)/24/3600" | bc
```

Когда bc не найден, выводится сообщение об ошибке, которое Zabbix интерпретирует как строку, хотя ожидает числовое значение.

Вот исправленная версия скрипта, которая не требует bc:

```bash
#! /bin/sh
#------------------------------------------------------------
# zext_ssl_cert.sh
# Script checks for number of days until certificate expires or the issuing authority
# depending on switch passed on command line.
#
#Based on script from aperto.fr (http://aperto.fr/cms/en/blog/15-blog-en/15-ssl-certificate-expiration-monitoring-with-zabbix.html)
#with additions by racooper@tamu.edu
#------------------------------------------------------------

DEBUG=0
if [ $DEBUG -gt 0 ]
then
    exec 2>>/tmp/my.log
    set -x
fi

f=$1
host=$2
port=$3
sni=$4
proto=$5

if [ -z "$sni" ]
then
    servername=$host
else
    servername=$sni
fi

if [ -n "$proto" ]
then
    starttls="-starttls $proto"
fi

case $f in
-d)
fix_broken_pipe=`openssl s_client -servername $servername -connect $host:$port -showcerts $starttls </dev/null 2>/dev/null |
          sed -n '/BEGIN CERTIFICATE/,/END CERT/p'`

end_date=`echo "$fix_broken_pipe" | openssl x509 -enddate -noout 2>/dev/null |
          sed -n 's/notAfter=//p' |
          sed 's/ GMT//g'`

if [ -n "$end_date" ]
then
    end_date_seconds=`date '+%s' --date "$end_date" 2>/dev/null`
    now_seconds=`date '+%s'`
    
    # Проверяем, что даты получены успешно
    if [ -n "$end_date_seconds" ] && [ "$end_date_seconds" -gt 0 ] 2>/dev/null
    then
        # Вычисляем разницу в днях без использования bc
        seconds_diff=$((end_date_seconds - now_seconds))
        days_diff=$((seconds_diff / 86400))
        echo "$days_diff"
    else
        # В случае ошибки возвращаем специальное значение
        echo "-1"
    fi
else
    echo "-1"
fi
;;

-i)
fix_broken_pipe=`openssl s_client -servername $servername -connect $host:$port -showcerts $starttls </dev/null 2>/dev/null |
          sed -n '/BEGIN CERTIFICATE/,/END CERT/p'`

issue_dn=`echo "$fix_broken_pipe" | openssl x509 -issuer -noout 2>/dev/null |
          sed -n 's/issuer=//p'`

if [ -n "$issue_dn" ]
then
    issuer=`echo $issue_dn | sed -n 's/.*CN=*//p'`
    if [ -n "$issuer" ]
    then
        echo "$issuer"
    else
        echo "Unknown"
    fi
else
    echo "Unknown"
fi
;;
*)
echo "usage: $0 [-i|-d] hostname port sni"
echo "    -i Show Issuer"
echo "    -d Show valid days remaining"
;;
esac
```

Основные изменения:

1. Убрана зависимость от bc - теперь используется встроенная арифметика shell ($((...)))
2. Добавлены проверки ошибок - если не удалось получить дату или произошла другая ошибка, возвращается "-1"
3. Исправлено вычисление дней - количество секунд в дне (86400) используется для прямого вычисления
4. Добавлена проверка корректности даты - проверяется, что end_date_seconds содержит допустимое значение

Теперь скрипт не требует установки bc и всегда возвращает числовое значение, которое Zabbix сможет корректно обработать.