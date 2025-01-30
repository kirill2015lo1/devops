

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
