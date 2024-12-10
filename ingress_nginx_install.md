Для того, чтобы работал ingress-nginx, перед его установкой, надо установить metallb, чтобы metallb мог выдавать ip для ingress, и мы могли получить доступ к нему

Вот ссылка на официальную документацию по установке:

https://kubernetes.github.io/ingress-nginx/deploy/

В данный момент установка через манифест, которую я использовал, выполняется так:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0-beta.0/deploy/static/provider/cloud/deploy.yaml
````
Чтобы изменить ip у ingress, который ему выдал metallb, на какой то один конкретный, который он всегда будет получать, например для того, 
чтобы с роутера сделать проброс портов с внешнего публичного ip роутера сразу в ingress, который уже будет направлять трафик напрямую в Pods, и следователь этот pod 
будет доступен напрямую в интернете, для этого надо:

Сначала открыть для редактирование service с типом Loadbalancer, и названием ingress-nginx-controller (название может быть чуть другое) который находится 
в namespace с названием ingress-nginx:
```
kubectl edit service/ingress-nginx-controller -n ingress-nginx
````
И далее нужно добавить, или отредактировать строку с параметром loadBalancerIP, и после двоеточия написать ip, который будет ему указывать от metallb, пример записи:

```
spec:
  loadBalancerIP: 192.168.50.135
```
Для понимая, самая простая схема развертывания с использова ingress, который будет перенаправлять нас в контайнер с образом nginx, выглядит так:

`pod <= service = <= ingress`

Как это работает:

Создается deployment, c образом nginx, далее создается обычный service, который привязан с подам, которые были созданы с использованием deployment, и далее создается ingress, 
который привязан к service.  
И теперь, при обращении к ingress, он через сервис, перенаправляет нас в поды

Теперь пример манифеста ingress, и пояснение с важным настройкам:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx3        
spec:
  ingressClassName: nginx   # Обязательно указываем имя класса, иначе ingress не будет работать 
  rules:
  - #host: test-test.com    # Это названием http заголовока host, который будет проверять ingress
    http:                          
      paths:
      - path: /katalog
        pathType: Prefix
        backend:
          service:
            name: nginx-service3   # название service, который привязан к нужным нам подам
            port:
              number: 80

```
Развернутые дополнение:

`ingressClassName: nginx` - без это настройки не будет работать ingress, он будет создан без класса, и не будет выполнять свою функция

`host: test-test.com` - здесь указывается http заголовок host, который будет слушать ingress, при указании заголовка, ingress будет выдавать ошибку при обращении к нему по ip, 
если нам нужен доступ к ingress по ip, например чтобы сделать port mapping с внешнего ip роутера на внутренний ip ingress, то эту строчку обязательно надо закоментировать! 

ВАЖНО! не стоит забывать, если host указан и не закоментировать, то доступ по указаному имени к ingress будет доступен только после добавления строчки в /etc/hosts, 
например если ip у ingress 192.168.50.135, а строчка с именем будет `host: test.com`, не забывайте добавить в /etc/hosts строчку: `192.168.50.135 test.com` 

`name: nginx-service3` - здесь указывается название service, который привязан к нужным нам подам, на которые мы хотим получить доступ

Далее идет пример большного манифеста, который сразу создаст configmap, deployment, service и ingress, и свяжен их между собой, сам ingress слушает будет слушать свой ip, 
так как строчка host закоментирована.

Доступ в pod будет через http://<ip ingress>/katalog, все что нужно, это подставить ip ingress, который ему выдал metallb

Сам манифест:
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: my-cm
data:
  default.conf: |
    server {
        listen 80 default_server;
        server_name _;

        charset utf-8;

        location / {
            default_type text/html;
            return 200 '
                <html>
                    <head><title>Success</title></head>
                    <body style="text-align: center; font-family: Arial, sans-serif; padding-top: 20%; font-size: 24px;">
                        <p>host: <strong>$host</strong></p>
                        <p><strong>У тебя все получилось</strong></p>
                        <p><strong>Это под с nginx, в который добавлен configmap с этим текстом</strong></p>
                    </body>
                </html>
            ';
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  nginx3
  namespace: default
  labels:
    app:  nginx3
spec:
  selector:
    matchLabels:
      app: nginx3
  replicas: 1
  template:
    metadata:
      labels:
        app:  nginx3
    spec:
      containers:
      - name:  nginx3
        image:  nginx
        ports:
        - containerPort:  80
        volumeMounts:
          - name:  config
            mountPath:  /etc/nginx/conf.d/
      volumes:
        - name:  config
          configMap:
            name: my-cm

---   
apiVersion: v1
kind: Service
metadata:
  name: nginx-service3
spec:
  
  selector:
    app: nginx3
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx3
spec:
  ingressClassName: nginx
  rules:
  - #host: test-test.com
    http:
      paths:
      - path: /katalog
        pathType: Prefix
        backend:
          service:
            name: nginx-service3
            port:
              number: 80

```

В моем случает ip у ingress 192.168.50.135, и поэтому получить html страницу я могу через:
```
curl 192.168.50.135/katalog
```
Вывод команды:
```
(venv) root@debian:~/study# curl 192.168.50.135/katalog

            <html>
                <head><title>Success</title></head>
                <body style="text-align: center; font-family: Arial, sans-serif; padding-top: 20%; font-size: 24px;">
                    <p>host: <strong>192.168.50.135</strong></p>
                    <p><strong>Y Tebia Bce poluchiloc !</strong></p>
                    <p><strong>Это под с nginx, в который добавлен configmap с этим текстом</strong></p>
                </body>
            </html>
```
