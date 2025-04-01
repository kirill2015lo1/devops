Нам нужен хост, который будет в качестве управление через ansible другими хостами, на главном хосте и на удаленных нодах должен быть установлен ssh

Потом качаем и устанавливаем ansible, инструкция на сайте тут:
```
sudo apt install wget gpg -y
```
https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html

Как скачали и установили выполняем команду для создание ключей ssh:
ssh-keygen
Она создаст приватный и публичный ключ для доступа по ssh

Далее копируем по очереди публичный ключ на каждый хост, которым мы хотим управлять через ansible:
```
ssh-copy-id root@192.168.50.110
```  
В данном случае мы скопировали ключ на хост с ip 192.168.50.110

Далее на главном хосте переходим в /etc/ansible/hosts:
```
nano /etc/ansible/hosts
```
Вот пример моего файлы hosts:
```
master1 ansible_host=192.168.50.110  #добавляем хосты, которыми мы хотим управлять
master2 ansible_host=192.168.50.111  
master3 ansible_host=192.168.50.112
worker1 ansible_host=192.168.50.120  #помещаем их в разные группы, чтобы можно
worker2 ansible_host=192.168.50.121  #было подавать команды только в определенные 
worker3 ansible_host=192.168.50.122  # и нужные нам группы
[kubernetes]  
master1
master2
master3
worker1
worker2
worker3
[masters]
master1
master2
master3
[workers]
worker1
worker2
worker3
[all:vars]
ansible_ssh_private_key_file=/root/.ssh/id_rsa
```

Тестовая команда для пинга всем подключенным хостам:
```
ansible all -m ping
```
