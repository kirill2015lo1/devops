Официальная документация rook

https://rook.io/docs/rook/latest-release/Getting-Started/quickstart/

В данный момент rook устанавливается так:
```
git clone --single-branch --branch v1.16.0 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml
```

На этой же странице есть подробное обьяснение с примера с создани своего хранилища
