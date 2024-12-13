Пример указания label для какого то узла:
```
kubectl label nodes <NODE_NAME> node-role.kubernetes.io/worker=worker
```
В данном случае будет ключ(key) - node-role.kubernetes.io/worker, а значение(value) - ""

Чтобы удалить label надо выполнить:
```
kubectl label nodes <NODE_NAME> node-role.kubernetes.io/worker-
```
Документация на использование affinity для pods:

https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector

Используется для указание deployment на каких узлах надо разворачивать pods, в данном случае на нодах с label, которые умеют название node-role.kubernetes.io/control-plane и со значением "", не 
буду создавать поды
```
kind: Deployment
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/control-plane  # Указать label, по которому будет идти отбор
                operator: NotIn     # In, NotIn, Exists, DoesNotExist, Gt и Lt
                values:
                - ""   # значение label
```
Конкретный пример, если при создании deployment мы укажем key: topology.kubernetes.io/zone, values: antarctica-east1 и antarctica-west1, то все Поды будут создаваться на узлах с label - topology.kubernetes.io/zone="antarctica-east1", или на узлах с label: topology.kubernetes.io/zone="antarctica-west1", но если из этих узлов будет узел с label - high: veryvery, все поды будут запускаться на нем, 
пока ресурсы не истекут, ведь preferredDuringSchedulingIgnoredDuringExecution указывает предпочтительный узел:
```
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - antarctica-east1
                - antarctica-west1
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1   #чем больше значение, тем выше приоритет для создание подов на этом узле
            preference:
              matchExpressions:
              - key: high
                operator: In
                values:
                - veryvery
```
Более легкие вариант, это nodeSelector, в нем просто можно указать label, и все pods будут создавать на узлах с этим label, пример:
```
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app:  nginx3
    spec:
      nodeSelector:
        node-role.kubernetes.io/control-plane: "" #указание ключа и значение для label
```
Документация на использование taint для Pods: 

https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

Для добавления taint на узел используйте команду kubectl taint:
```
kubectl taint nodes <node-name> <key>=<value>:<effect>
````
Пример:
```
kubectl taint nodes worker1 node-role.kubernetes.io/worker=NoSchedule
```
Здесь: 

`node-role.kubernetes.io/worker` — это ключ taint.

`NoSchedule` — это эффект (значение taint), который определяет, что на узле нельзя запускать поды, если они не имеют соответствующую toleration.

Другие возможные эффекты taint:

`NoSchedule`: Запрещает планирование подов на узле, если под не имеет соответствующего toleration.

`PreferNoSchedule`: Kubernetes предпочтительнее не будет планировать поды на этом узле, но будет планировать, если нет других вариантов.

`NoExecute`: Помимо того, что запрещает планирование, также удаляет поды с этого узла, если они не имеют соответствующего toleration.

Чтобы удалить taint с узла, используйте команду:
```
kubectl taint nodes <node-name> <key>:<effect>-
```

Пример:
```
kubectl taint nodes worker1 node-role.kubernetes.io/worker:NoSchedule-
```
Эта команда удаляет taint `node-role.kubernetes.io/worker=worker:NoSchedule` с узла `worker1`, что позволяет подам снова быть запланированными на этом узле без необходимости наличия соответствующего toleration.

Как указать Toleration в Deployment, чтобы поды могли быть размещены на узле с taint, нужно указать соответствующий toleration в манифесте Deployment или Daemonset.

Пример с toleration для taint в Deployment:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
        - name: example-container
          image: nginx
      tolerations:
        - key: "node-role.kubernetes.io/worker"
          effect: "NoSchedule"  #будет толерантен к указанному ключу с любым значением
```
Но есть нам нужно чтобы была толерантность была к любым ключам и значениям, но только если у них effect: "NoSchedule" (полезно для сброра метрик), то мы меняем key на operator: exists :
```
      tolerations:
        - operator: "Exists"    
          effect: "NoSchedule"  #будет толерантен к любому ключу с любым значением с таким эффектом
```
