Ссылка на документацию по распределению ресурсов для расвертываний:

https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/ - CPU

https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/ - RAM

Пример указание ресурсов, `requests` - это ресурсы которые будут зарезервированны для этого сервиса, `limits` - это граница, если pod ее превысит, то будет перезапущен:
```
spec:
  template:
    spec:
      containers:
      - name:  nginx3
        image:  nginx
        ports:
        - containerPort:  80
        resources:
          limits:
            memory: "1Gi"
            cpu: "1"
          requests:
            memory: "100Mi"
            cpu: "0.100"
```

---
Ссылка на документацию по самопроверкам подов:

https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

Пример простого deployment с readinessProbe(если проверка не пройдена, трафик перестает поступать на этот Pod, и перенаправляется на другие здоровые поды), 
и livenessProbe(если проверка не пройден, под будет перезапущен)
```
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
  replicas: 3
  template:
    metadata:
      labels:
        app:  nginx3
    spec:
      containers:
      - name:  nginx3
        image:  nginx
        ports:
        - containerPort:  80 #по какому порту идет http проверка 
        readinessProbe:    #Если общая проверка не пройдена, трафик перестанет идти в pod
          failureThreshold: 2 #Если проверка не пройдена 2 раз подряд, то общая проверка провалена
          httpGet:
            path: /
            port: 80
          periodSeconds: 5  #Раз во сколько секунд выполняется проверка
          successThreshold: 1 # Сколько раз надо пройти проверку, чтобы считалось что все ок
          timeoutSeconds: 1 #Время на ответ пода на проверку
          initialDelaySeconds: 5 #Дилей 5 секунд перед выполнением первой проверки.
        livenessProbe:     # Если общая проверка не пройдена, pod будет перезапущен
          failureThreshold: 2
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
          initialDelaySeconds: 5
```
