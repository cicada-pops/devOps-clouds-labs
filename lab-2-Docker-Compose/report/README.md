# Лабораторная №2* 

***"Плохой" файл был искуственно сгенерирован, а затем я нашел проблемы и исправил их!**

(Хороший и плохой варианты файлов выложены в репозиторий)

## Плохие практики в docker-compose 

1. Жесткая привязка портов хостовой машины к контейнерам
-  Плохая практика: В файле явно указаны привязки портов к контейнерам, а это риск конфликтов с другими сервисами на нашем хосте, использующими те же порты.

2. Неправильное использование зависимостей между сервисами (depends_on)
- Плохая практика: Использование depends_on не гарантирует, что сервис полностью запущен и готов к работе. Он указывает только на порядок запуска контейнеров, но не проверяет готовность базы.

3. Привязка конфиденциальных данных в открытых настройках
- Плохая практика: В environment переменные среды содержат сереты, которые хранятся прямо в файле конфигурации. (Никто же не хочет получить по башке от бати, что ему на почту приходит всякая жуть...)

## Как исправить?

1. Исправление жесткой привязки портов
- Вместо привязки конкретных портов, в "хорошем" файле используем другие порты.

2. Исправление механизма зависимостей
- Использум топовый механизм healthcheck для бдшки. Вместо простого depends_on, теперь бд будет проверяться на готовность перед тем, как другие контейнеры начнут с ней взаимодействовать.

3. Исправление хранения конфиденциальных данных
- Используются Docker Secrets для передачи пароля базы данных, теперь батя будет спать спокойно <3.


### Изоляция контейнеров внутри проекта

Для изоляции сервисов друг от друга можно воспользоваться различными сетями. Это позволит контейнерам быть частью одной сборки, но они не будут иметь возможность взаимодействовать напрямую через сеть, если находятся в разных сетях.

Пример использования:

```
version: '3.8'

services:
  web:
    image: nginx:stable
    ports:
      - "8080:80"
    environment:
      - ENV=${ENVIRONMENT}
    volumes:
      - ./app:/usr/share/nginx/html:ro
    networks:
      - web_net

  db:
    image: postgres:14
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/postgres_password
      - POSTGRES_DB=mydb
    secrets:
      - postgres_password
    ports:
      - "5433:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
    networks:
      - db_net

  cache:
    image: redis:6-alpine
    ports:
      - "6378:6379"
    networks:
      - cache_net

volumes:
  db_data:

secrets:
  postgres_password:
    file: ./secrets/postgres_password

networks:
  web_net:
    driver: bridge
  db_net:
    driver: bridge
  cache_net:
    driver: bridge
```

- Создаем три сети: web_net, db_net и cache_net. При этом каждый сервис подключён к своей сети.

Docker использует bridge по умолчанию для контейнеров в Compose. Если сервисы находятся в разных сетях, они изолированы и не могут обмениваться данными друг с другом, если только специально не настроить маршруты или не добавить их в общую сеть.