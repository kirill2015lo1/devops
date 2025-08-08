Чтение:
```
dd if=/dev/sdX of=/dev/null bs=1M count=10240 iflag=direct  
```
Запись:
```
dd if=/dev/urandom of=/dev/sdX bs=1M count=10240 oflag=direct status=progress 
```
