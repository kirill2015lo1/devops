Заходим на хост, с которого будет установка k8s через kubespray, в моем случае это VM с debian 12, все команды будут происходить только с него

По хорошему на нем уже заранее должны быть созданы ключи через `ssh-keygen`, и скопированны через `ssh-copy-id`,  на хосты, которые мы хоти поместить в кластер k8s
Лучше изначально настроить ansible на этом же хосте, ведь он нуждается вышеперечисленных вещах

Видео для визаульного понимаю установки:  
https://www.youtube.com/watch?v=lvkpIoySt3U&t=1246s

Заранее лучше через ansible выполнить команду, чтобы днс сервер на всех нодах, которые мы будет добавлять в кластер k8s был 8.8.8.8, это нужно для скачивания файлов при установке kubespray, команда такая:
```
ansible all -m command -a 'apt install sudo -y'
ansible all -m command -a 'sh -c "echo nameserver 8.8.8.8 > /etc/resolv.conf"' --become
```
На управляющем хосте скачиваем git:
```
apt install git -y
```
Скачиваем и настраиваем ansible, гайд по настроке в этом же репозитории 

Скачиваем python3 и python3-venv:
```
apt install python3 -y
apt install python3-venv
```
Создаем venv в папке в опередленной папке:
python3 -m venv (здесь название папки)
В моем случае это папка будет такая:
```
python3 -m venv /root/study/venv
```
Далее активируем виртуальное окружение:
```
source /root/study/venv/bin/activate
```
Далее переходим в директорию /root/study и копируем в нее репозиторий с kubespray:
```
cd /root/study/
git clone https://github.com/kubernetes-sigs/kubespray.git
git checkout tags/(название версии kubespray, которую вы хотим использовать), посмотреть версии можно в гитхабе в релизах, в моем случае v2.23.0
git checkout tags/v2.23.0
```
Переходим в скачанную директорию kubespray:
```
cd kubespray
```
Устанавливаем в venv все зависимости, которые нужны kubespray:
```
pip install -U -r requirements.txt
```
Копируем директорию со стандартными настройки в отдельную директорию с названием mycluster, и уже ее будем настраивать
```
cp -rfp inventory/sample inventory/mycluster
```
Указываем ip своих нод, которые будут в кластере kuberntes 
declare -a IPS=(здесь в скобках перечислить все ip нод через пробел), в моем случае будет так:
```
declare -a IPS=(192.158.50.110 192.168.50.111 192.168.50.112 192.168.50.120 192.168.50.121 192.168.50.122)
```
И записываем все эти ip в файлик host.yaml, через скрипт в директории contrib/inventory_builder/inventory.py ${IPS[@]}:
```
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]} 
```
Далее переходим открываем только что созданный файл, и его редактируем под свои нужды:
```
nano inventory/mycluster/hosts.yaml 
```
Готовый файл у меня получился таким:
```
all:
  hosts:
    master1:
      ansible_host: 192.158.50.110
      ip: 192.158.50.110
      access_ip: 192.158.50.110
    master2:
      ansible_host: 192.168.50.111
      ip: 192.168.50.111
      access_ip: 192.168.50.111
    master3:
      ansible_host: 192.168.50.112
      ip: 192.168.50.112
      access_ip: 192.168.50.112
    worker1:
      ansible_host: 192.168.50.120
      ip: 192.168.50.120
      access_ip: 192.168.50.120
    worker2:
      ansible_host: 192.168.50.121
      ip: 192.168.50.121
      access_ip: 192.168.50.121
    worker3:
      ansible_host: 192.168.50.122
      ip: 192.168.50.122
      access_ip: 192.168.50.122
  children:
    kube_control_plane:
      hosts:
        master1:
        master2:
        master3:
    kube_node:
      hosts:
        master1:
        master2:
        master3:
        worker1:
        worker2:
        worker3:
    etcd:
      hosts:
        master1:
        master2:
        master3:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```
И далее надо подредактировать парметры самого кластера, ко которым он будет устанавливаться, и подредактироват addons, дополнения, которые будут установлены
`/kubespray/inventory/mycluster/group_vars/k8s_cluster/addons.yml` - здесь лежат адоны
`/kubespray/inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`  - здесь лежат настройки кластера 

Возможна ошибка при установке kubespray v2.23.0, команда которая решает эту проблему:
```
pip install ansible-core==2.14.11
```
Ссылка на обсуждение, где увидел решение:
https://github.com/kubernetes-sigs/kubespray/issues/10688

Далее переходим обратно в директорию kubespray и выполняем установку:
```
cd /root/study/kubespray
ansible-playbook -i inventory/mycluster/ -u root -b -v --private-key=~/.ssh/id_rsa cluster.yml
```
Если вылезает какая то ошибка, и описание ошибки при установке скрыто, то надо открыть файл all.yml и поменять чтроку unsafe_show_logs с false на true, путь к файлу:
```
nano /inventory/mycluster/group_vars/all/all.yml
```
Обсуждение, где решение этой проблемы:  
https://github.com/kubernetes-sigs/kubespray/issues/9037

Для того, чтобы можно было управлять кластером с управляющего хоста хоста, который не является master нодой, и не находится в кластере, надо выполнить команду:
```
ansible master1 -m command -a 'cat /etc/kubernetes/admin.conf'
```
Скопировать все выведеное содержимое и встав по пути ~/.kube/config, для этого надо:
```
mkdir /root/.kube/
nano /root/.kube/config
```
И сохраняем туда, что скопировали до этого, и строчку `server: https://127.0.0.1:6433` меняем на ip хоста, c которого скопировали содержимое admin.conf, в моем случае:
```
server: https://192.168.50.110:6443
```
Далее надо скачать саму команду kubectl, `https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/` ссылка на документацию по скачиванию, в моем случае:
```
apt install curl -y
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
Проверка установки kubectl:
```
kubectl version --client
```
Проверка правильной настройки kubectl для управлений кластером kubernetes:
```
kubectl get nodes
```
Настройки autocomplition:
```
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc
source /etc/bash_completion
```
Сделать редактор nano по умолчанию:
```
export VISUAL=nano
export EDITOR="$VISUAL"
```
