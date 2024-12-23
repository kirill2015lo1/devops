Официальная документация rook:

https://rook.io/docs/rook/latest-release/Getting-Started/quickstart/

Ссылка где можно почитать и 3 типах хранилищах, которые будут доступны для создания, чтобы храненить информацию, с приложений, 
запущенный в кластере k8s:

https://rook.io/docs/rook/latest-release/Getting-Started/quickstart/#create-a-ceph-cluster

В данный момент rook устанавливается так:
```
git clone --single-branch --branch v1.16.0 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml
```


