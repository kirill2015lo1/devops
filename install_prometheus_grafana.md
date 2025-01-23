Самый простой способ установки, это установка helm chart от Prometheus Community, под названиием kube-prometheus-stack    
Ссылка на github с инструкцией по установке:   
https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

Ссылка на видео с подробный обьяснением обьяснением по установке и использованию:  
https://www.youtube.com/watch?v=6xmWr7p5TE0&t=90s  
Обьяснение работы prometheus в docker, многи моменты схожи при работе в kubernetes:  
https://www.youtube.com/watch?v=Q_fKb0nrfCg&t=2425s  

Установка сейчас (Производится с помощью helm, поэтому он должен быть установлен заранее):
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
Далее:
```
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```
`--namespace monitoring` указывает в каком namespace нужно установить,   
`--create-namespace` создаст этот namespace, если его до этого не было  
`prometheus` это под каким названием будет установлен этот helm chart 

Чтобы получить доступ к веб интерфесу prometheus и grafana, нужно в namespace под названием monitoring  
Изменить типы следующих сервисов c ClusterIp на LoadBalancer:   
`service/prometheus-kube-prometheus-prometheus`  
`service/prometheus-grafana `

Стандартные данные для входа для grafana:
user: admin  
pass: prom-operator  

По стандарту prometheus не сможет подключиться к метрикам kube-proxy, вот решение: 
https://stackoverflow.com/questions/60734799/all-kubernetes-proxy-targets-down-prometheus-operator


Далее если приложение отдает метрики, и надо настроить, чтобы prometheus стал забирать эти метрики, надо создать servicemonitor,  
И привязать его к service приложения, пример:
```
---
apiVersion: v1
kind: Service
metadata:
  name: flask
  labels:
    job: test
    app: flask
spec:
  selector:
    app: flask
  ports:
    - name: web
      protocol: TCP
      port: 80
      targetPort: 5000
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: flask
  labels:
    release: prometheus
spec:
  jobLabel: job #необязательно, добавляет к метрикам в prometheus такое параметр, чтобы легче было искать метрики
  selector:
    matchLabels:
      app: flask  #Начнем забирать метрики через service с таким label
  endpoints:
  - port: web
    interval: 30s
    path: /metrics  
```
