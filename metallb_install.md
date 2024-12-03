Ссылка на хорошую статью на github, с хорошим обьяснение настройки:
https://github.com/morrismusumi/kubernetes/tree/main/clusters/homelab-k8s/apps/metallb-plus-nginx-ingress

Ссылка на официальный сайт, для установки metallb: 
https://metallb.io/installation/


Для начала надо установить metallb внутри кластера kubernetes, он будет раздвать ip адреса:
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

Проверить установку можно командой:
```
kubectl get all -n metallb-system
kubectl api-resources | grep metallb
```
Чтобы matallb работал надо запустить:
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.50.120-192.168.50.130 #Тут пул адресов, который будет раздаваться loadbalancer service 
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```
После этого адреса будут раздавать в указанном диапазоне

При создании loadbalacner можно указать, какой ip он будет получать:
```
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer
  namespace: default
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.50.xxx # Укажите здесь нужный IP-адрес, который он будет получать
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: my-app
```
