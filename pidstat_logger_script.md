Скрипт для добавление логера, который будет в `/var/log/iotop/iotop.log` писать процессы, которые потребляют больше всего диска
```
apt install sysstat -y
cat > /usr/local/bin/iotop-logger.sh <<EOF
#!/bin/bash
mkdir -p /var/log/iotop
while true; do
  linecount=\$(wc -l < /var/log/iotop/iotop.log)
  if [ \$linecount -gt 150000 ]; then
    sed -i '1,1000d' /var/log/iotop/iotop.log
  fi
  pidstat -d 10 10 >> /var/log/iotop/iotop.log
  sleep 10
done
EOF
chmod +x /usr/local/bin/iotop-logger.sh
cat > /etc/systemd/system/iotop-logger.service <<EOF2
[Unit]
Description=IOTop Disk IO Logger
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/bin/iotop-logger.sh
Restart=always
RestartSec=10
User=root
[Install]
WantedBy=multi-user.target
EOF2
systemctl daemon-reload
rm -R /var/log/iotop/iotop.log
systemctl start iotop-logger.service
systemctl restart iotop-logger.service
systemctl status iotop-logger.service
```
