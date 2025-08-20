Команда 
```
(zcat /var/log/syslog.*.gz 2>/dev/null; cat /var/log/syslog* 2>/dev/null; zcat /var/log/dpkg.log.*.gz 2>/dev/null; cat /var/log/dpkg.log* 2>/dev/null) | grep --text -i failed
```
