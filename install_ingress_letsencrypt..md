Ссылка на документацию cert-manamger

https://cert-manager.io/docs/

Ссылка на видео с хорошим обьяснением:

https://www.youtube.com/watch?v=N7W_nsEA-Ao

https://www.youtube.com/watch?v=DJ2sa49iEKo

В нашем случае установка будет через yaml файла, ссылка на подробный гайд:

https://cert-manager.io/docs/installation/kubectl/

В данным момент делается так, сначала устанавливаем:
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```
В данном манифесте будет установка CRDs(самописные обьекты в k8s), и сразу созданы нужные развертования

Далее создаем обьект, который будет ходить и обновлять сертификаты на указанные нами ресурсы (ACME):
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-isseur #Указываем название для isseur
  namespace: cert-manager #Этот NS будет создан при установке cert-manager
spec:
  acme:
    email: tsurankirill@mail.ru #Указывем нашу почту
    server: https://acme-v02.api.letsencrypt.org/directory #ссылка на ресурс
    privateKeySecretRef:
      name: issuer-key1 #Указываем название для secret
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx #указываем class у ingress
```

`https://acme-v02.api.letsencrypt.org/directory` - ссылку мы взяли с официального сайта letsencrypt https://letsencrypt.org/getting-started/

`name: letsencrypt-isseur` и `name: issuer-key1` далее укажем в ingress 

Далее создаем ingress и указываем параметры для isseur, которые указали до этого, пример ingress:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations: #обязательно добавляем анотоцию, чтобы 
    cert-manager.io/cluster-issuer: letsencrypt-isseur
  name: ingress-nginx3
spec:
  ingressClassName: nginx
  rules:
  - host: testsmth.ru #ниже укажем хост
    http:
      paths:
      - path: /katalog
        pathType: Prefix
        backend:
          service:
            name: nginx-service3
            port:
              number: 80
  tls: 
  - hosts:
    - testsmth.ru #указываем хост, на который будет распространяться tls
    secretName: issuer-key1 #Указываем имя secret в isseur
```
После применения этого ingress, подключение к поду, к которому ведет ingress будет уже по https
