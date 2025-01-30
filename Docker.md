

services:
  prometheus: #server
    image: prom/prometheus # Имя образа, можно заменить на нужный
    ports:
      - "9090:9090"  # Проброс порта (локальный:контейнерный)
    #environment:
      #var: value2   # Переменные окружения
    volumes:
      - ../configs/prometheus.yaml:/etc/prometheus/prometheus.yaml  # Монтирование локальной папки в контейнер
      - prometheusdata:/prometheus  # Хранение данных вне контейнера
    restart: always  # Автоматический рестарт при сбое


  grafana:
    image: grafana/grafana  # Образ PostgreSQL
    #environment:
      #var: value2   # Переменные окружения
    volumes:
      - grafanadata:/var/lib/grafana  # Хранение данных вне контейнера
    restart: always  # Автоматический рестарт при сбое
    depends_on:
      - prometheus  # Контейнер запускается после db
    ports:
      - "3000:3000"  # Проброс порта (локальный:контейнерный)


volumes:
  prometheusdata:  # Определение именованного тома для базы данных
  grafanadata:

wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvf node_exporter-1.8.2.linux-amd64.tar.gz
cd node_exporter-1.8.2.linux-amd64
sudo cp node_exporter /usr/local/bin/

 
docker compose -f prom-graphana.yaml up -d 
docker compose -f prom-graphana.yaml down

Если у вас docker-compose.yml называется по-другому, например my-compose.yml, то при запуске нужно указать его явно с флагом -f:

```
docker compose -f my-compose.yml up

````
установка для debian
https://docs.docker.com/engine/install/debian/
```
version: '3.8'  # Версия Docker Compose

services:
  app: #server
    image: myapp:latest  # Имя образа, можно заменить на нужный
    build:
      context: .  # Контекст сборки (текущая папка)
      dockerfile: Dockerfile  # Имя Dockerfile (по умолчанию)
    ports:
      - "8080:80"  # Проброс порта (локальный:контейнерный)
    environment:
      - DATABASE_URL=postgres://user:password@db:5432/mydatabase  # Переменные окружения
    volumes:
      - ./app:/usr/src/app  # Монтирование локальной папки в контейнер
    depends_on:
      - db  # Контейнер запускается после db

  db:
    image: postgres:15  # Образ PostgreSQL
    restart: always  # Автоматический рестарт при сбое
    environment:
      POSTGRES_USER: user  # Имя пользователя
      POSTGRES_PASSWORD: password  # Пароль
      POSTGRES_DB: mydatabase  # Имя базы данных
    volumes:
      - pgdata:/var/lib/postgresql/data  # Хранение данных вне контейнера


volumes:
  pgdata:  # Определение именованного тома для базы данных
```
