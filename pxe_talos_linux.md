
На клиенте-бастионе генерим ключ
```
ssh-keygen -t ed25519
```
Выводим и копируем 
```
cat ~/.ssh/id_ed25519.pub
```

Вставляем ключ на сервере с sshd
```
ssh root@192.168.0.
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

Установка докера
``` 
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```
Создаем директорию и качаем бинарники для dhcp:
```
cd /
mkdir pxe-talos
cd pxe-talos
sudo curl -L -o /pxe-talos/undionly.kpxe  https://boot.ipxe.org/undionly.kpxe
sudo curl -L -o /pxe-talos/ipxe.efi      https://boot.ipxe.org/ipxe.efi
```

Качаем talos 
```
sudo curl -L -o vmlinuz-amd64 \
  https://github.com/siderolabs/talos/releases/latest/download/vmlinuz-amd64
sudo curl -L -o initramfs-amd64.xz \
  https://github.com/siderolabs/talos/releases/latest/download/initramfs-amd64.xz
```

https://cozystack.io/docs/install/talos/pxe/



echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
