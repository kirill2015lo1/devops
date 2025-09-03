
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

Примеры cozystack:
```
https://cozystack.io/docs/install/talos/pxe/
```

Ссылка на релизы талоса:
```
https://github.com/siderolabs/talos/releases
```

```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

```
sudo docker run --name=dnsmasq -d --cap-add=NET_ADMIN --net=host quay.io/poseidon/dnsmasq:v0.5.0-32-g4327d60-amd64 \
  -d -q -p0 \
  --dhcp-range=192.168.100.200,192.168.100.205 \
  --dhcp-option=option:router,192.168.100.1 \
  --enable-tftp \
  --tftp-root=/var/lib/tftpboot \
  --dhcp-match=set:bios,option:client-arch,0 \
  --dhcp-boot=tag:bios,undionly.kpxe \
  --dhcp-match=set:efi32,option:client-arch,6 \
  --dhcp-boot=tag:efi32,ipxe.efi \
  --dhcp-match=set:efibc,option:client-arch,7 \
  --dhcp-boot=tag:efibc,ipxe.efi \
  --dhcp-match=set:efi64,option:client-arch,9 \
  --dhcp-boot=tag:efi64,ipxe.efi \
  --dhcp-userclass=set:ipxe,iPXE \
  --dhcp-boot=tag:ipxe,http://192.168.100.180:8080/boot.ipxe \
  --log-queries \
  --log-dhcp
```

```
TALOS_VER="1.10.7"
mkdir -p /pxe-talos/matchbox_image
sudo chown $(id -u):$(id -g) /pxe-talos/matchbox_image -R
cd /pxe-talos/matchbox_image
curl -L -o /pxe-talos/matchbox_image/vmlinuz \
  "https://github.com/siderolabs/talos/releases/download/v${TALOS_VER}/vmlinuz-amd64"
curl -L -o /pxe-talos/matchbox_image/initramfs.xz \
  "https://github.com/siderolabs/talos/releases/download/v${TALOS_VER}/initramfs-amd64.xz"
cat << 'EOF' > /pxe-talos/matchbox_image/talos-default.json
{
  "id": "default",
  "name": "default",
  "boot": {
    "kernel": "/assets/vmlinuz",
    "initrd": ["/assets/initramfs.xz"],
    "args": [
      "initrd=initramfs.xz",
      "init_on_alloc=1",
      "slab_nomerge",
      "pti=on",
      "console=tty0",
      "printk.devkmsg=on",
      "talos.platform=metal"
    ]
  }
}
EOF

cat << 'EOF' > /pxe-talos/matchbox_image/group-default.json
{
  "id": "default",
  "name": "default",
  "profile": "default"
}
EOF

docker run -d --name matchbox --net host \
  -v /pxe-talos/matchbox_image/initramfs.xz:/var/lib/matchbox/assets/initramfs.xz:ro \
  -v /pxe-talos/matchbox_image/vmlinuz:/var/lib/matchbox/assets/vmlinuz:ro \
  -v /pxe-talos/matchbox_image/talos-default.json:/var/lib/matchbox/profiles/default.json:ro \
  -v /pxe-talos/matchbox_image/group-default.json:/var/lib/matchbox/groups/group-default.json:ro \
  quay.io/poseidon/matchbox:latest \
  -address=:8080 \
  -log-level=debug
```

### Пример патча для patch_master1.yaml
```
machine:
  network:
    interfaces:
      - interface: ens19
        addresses:
          - 192.168.100.101/24
        vip:
          ip: 192.168.100.110
    nameservers:
      - 192.168.100.240
      - 192.168.0.240
      - 8.8.8.8
      - 1.1.1.1
    searchDomains:
      - tsuran.local
  nodeLabels:
    node.kubernetes.io/exclude-from-external-load-balancers: ""
    lol.kek: "test-master"
```
### Пример патча для patch_worker1.yaml
```
machine:
  network:
    interfaces:
      - interface: ens19
        addresses:
          - 192.168.100.111/24
    nameservers:
      - 192.168.100.240
      - 192.168.0.240
      - 8.8.8.8
      - 1.1.1.1
    searchDomains:
      - tsuran.local
  nodeLabels:
    node.kubernetes.io/exclude-from-external-load-balancers: ""
    lol.kek: "test-worker"
```

### секреты генерим 
```
talosctl gen secrets -o secrets.yaml
```

### конфиги генерим 
```
talosctl gen config --kubernetes-version 1.31.4 --with-secrets secrets.yaml my-cluster https://192.168.100.150:6443
```

### Для мастер нод
```
talosctl machineconfig patch controlplane.yaml --patch @patch_master.yaml --output master1.yaml
talosctl machineconfig patch controlplane.yaml --patch @patch_master.yaml --output master2.yaml  
talosctl machineconfig patch controlplane.yaml --patch @patch_master.yaml --output master3.yaml
talosctl machineconfig patch controlplane.yaml --patch @patch_master.yaml --output master4.yaml
talosctl machineconfig patch controlplane.yaml --patch @patch_master.yaml --output master5.yaml  
```

### Для воркер нод
```
talosctl machineconfig patch worker.yaml --patch @patch_worker.yaml --output worker1.yaml
talosctl machineconfig patch worker.yaml --patch @patch_worker.yaml --output worker2.yaml
talosctl machineconfig patch worker.yaml --patch @patch_worker.yaml --output worker3.yaml
talosctl machineconfig patch worker.yaml --patch @patch_worker.yaml --output worker4.yaml
talosctl machineconfig patch worker.yaml --patch @patch_worker.yaml --output worker5.yaml
```

### юзаем на ноды мастер
```
talosctl apply-config --insecure -n 192.168.100. --file ./master1.yaml
talosctl apply-config --insecure -n 192.168.100. --file ./master2.yaml
talosctl apply-config --insecure -n 192.168.100. --file ./master3.yaml
talosctl apply-config --insecure -n 192.168.100. --file ./master4.yaml
talosctl apply-config --insecure -n 192.168.100. --file ./master5.yaml

```
### юзаем на ноды мастер
```
talosctl apply-config --insecure -n 192.168.100. --file ./worker1.yaml
talosctl apply-config --insecure -n 192.168.100. --file ./worker2.yaml
talosctl apply-config --insecure -n 192.168.100. --file ./worker3.yaml
talosctl apply-config --insecure -n 192.168.100. --file ./worker4.yaml
talosctl apply-config --insecure -n 192.168.100. --file ./worker5.yaml
```



### юзаем бутстрап на любую мастер ноду
```
talosctl bootstrap -n 192.168.100.101 -e 192.168.100.101 --talosconfig=./talosconfig
```

### с нее же забираем кубконфиг с vip и направляем кубконфиг на этот же vip 
```
talosctl kubeconfig ~/.kube/config -n 192.168.100.150 -e 192.168.100.150 --talosconfig ./talosconfig
```

### gping
```
gping 192.168.100.101 192.168.100.102 192.168.100.103 192.168.100.111 192.168.100.112 192.168.100.113 192.168.100.150 
```


### как посмотреть резолвы 
```
talosctl get resolvers -n 192.168.100.101 -e 192.168.100.150 --talosconfig=./talosconfig
```
