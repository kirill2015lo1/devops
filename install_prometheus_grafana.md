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

Если нужны pvc для:  
statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus  

Если helm chart еще не установлен, то сохраняем values:
```
helm show values prometheus-community/kube-prometheus-stack > values.yaml
```
Открываем values.yaml, для prometheus ищем `prometheusSpec.storageSpec` и заполняем по примеру:
```
    storageSpec: 
      volumeClaimTemplate:
        metadata:
          name: prometheus-data
        spec:
          storageClassName: rook-cephfs
          accessModes: ["ReadWriteMany"]
          resources:
            requests:
              storage: 10Gi
```
Для alertmanager ищем alertmanagerSpec.storage и заполняем по такому же примеру:
```
    storage: 
      volumeClaimTemplate:
        metadata:
          name: alertmanager-data
        spec:
          storageClassName: rook-cephfs
          accessModes: ["ReadWriteMany"]
          resources:
            requests:
              storage: 5Gi
```
Еще можно сократить дней, скольок хранятся метрики, и их максимальный размер:
```
    retention: 10d

    retentionSize: "7Gi"
```
deployment grafana, через edit меняем на statefulset и заполняем хранилище по примеру:
```
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      creationTimestamp: null
      name: storage
    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 5Gi
      storageClassName: rook-cephfs
      volumeMode: Filesystem
```


Далее полняем команду:
```
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace -f values.yaml
```
Теперь, если все сделано нормально, можно посмотреть, будет два pvc:
```
(venv) root@debian:~/study/flask_app_k8s# kubectl get pvc -n monitoring 
NAME                                                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
alertmanager-data-alertmanager-prometheus-kube-prometheus-alertmanager-0   Bound    pvc-83293887-866d-4043-af19-45793693c9a2   5Gi        RWX            rook-cephfs    <unset>                 21m
prometheus-data-prometheus-prometheus-kube-prometheus-prometheus-0         Bound    pvc-5d0de89c-b93c-42f2-8dde-76744e0f09ad   10Gi       RWX            rook-cephfs    <unset>                 22m
storage-prometheus-grafana-0                                               Bound    pvc-9f94ab5f-8aac-4a44-bd35-b9bb5f21be43   5Gi        RWX            rook-cephfs    <unset>                 7m42s
```

Если helm chart уже установлен, но перед этим pvc не были указаны, то сначала запрашиваем текущие values:
```
helm get values prometheus -n monitoring --all -o yaml > values.yaml
```
Дале правим prometheusSpec.storageSpec и alertmanagerSpec.storage по примеру выше, и выполняем команду для применения изменений:
```
helm upgrade prometheus prometheus-community/kube-prometheus-stack --namespace monitoring -f values.yaml
```
Все готово, далее: 

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





Небольшая шпора по promql:  
https://stepik.org/lesson/1305857/step/2?unit=1320830
