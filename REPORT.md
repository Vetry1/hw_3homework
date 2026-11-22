# Как я проверял счётчик

## Один контейнер

Сначала собрал образ и запустил его с приветствием `Hello from Docker!`. Для этой проверки взял порт 8081.

```sh
docker build -t hw3-counter .
docker run -d --name counter-single -p 127.0.0.1:8081:8000 \
  -e GREETING='Hello from Docker!' hw3-counter

url=http://127.0.0.1:8081
curl --retry 8 --retry-all-errors "$url/"
curl -X POST "$url/increment"
curl -X POST "$url/increment"
docker logs counter-single
```

При первом запросе в ответе было `"greeting": "Hello from Docker!"` и `"count": 0`. После двух POST счётчик показал сначала 1, потом 2. Вот эти же запросы в логах контейнера:

```text
2026-11-22 17:25:39,372 listening port=8000 storage=sqlite
2026-11-22 17:25:40,356 http 172.17.0.1 "GET / HTTP/1.1" 200 -
2026-11-22 17:25:40,367 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
2026-11-22 17:25:40,373 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
```

Перед следующей проверкой освободил порт:

```sh
docker stop counter-single
docker rm counter-single
```

## SQLite и том

Для SQLite использовал `compose.sqlite.yaml`. В нём к папке `/data` подключён том `sqlite-data`, туда приложение сохраняет файл базы.

```sh
cp .env.example .env
export APP_PORT=8081
docker compose -p counter-sqlite -f compose.sqlite.yaml up -d --build
url=http://127.0.0.1:8081
curl --retry 8 --retry-all-errors "$url/"
curl -X POST "$url/increment"
```

Сначала было 0, после POST стало 1. Затем удалил контейнер и запустил его заново:

```sh
docker compose -p counter-sqlite -f compose.sqlite.yaml down
docker compose -p counter-sqlite -f compose.sqlite.yaml up -d
curl --retry 8 --retry-all-errors "$url/"
```

В ответе поменялся `hostname`: был `1afd4c6203c0`, стал `5e464edb8ae2`. Значение осталось 1. После проверки остановил этот вариант и вернул порт из `.env`:

```sh
docker compose -p counter-sqlite -f compose.sqlite.yaml down
unset APP_PORT
```

## Redis

В основном `compose.yaml` приложение подключается к Redis по адресу `redis://redis:6379/0`. Запустил оба контейнера и посмотрел их состояние:

```sh
docker compose up -d --build --wait
docker compose ps
```

У обоих был статус `healthy`. Порт 8080 на компьютере был открыт только у приложения.

```sh
url=http://127.0.0.1:8080
curl "$url/"
curl -X POST "$url/increment"
curl -X POST "$url/increment"
curl -X POST "$url/increment"
```

Начал с 0. Три запроса подряд дали 1, 2 и 3. Проверил, что будет с этой тройкой после пересоздания контейнеров:

```sh
docker compose down
docker compose up -d --wait
curl "$url/"
```

Ответ после запуска:

```json
{"greeting": "Привет из Docker!", "count": 3, "hostname": "916642e9fbbe", "storage": "redis"}
```

### Что происходит, если остановить Redis

Остановил только Redis. Добавил к `curl` вывод HTTP-кода, чтобы увидеть, какие запросы всё ещё работают.

```sh
docker compose stop redis
curl -s -w ' HTTP %{http_code}\n' "$url/healthz"
curl -s -w ' HTTP %{http_code}\n' "$url/readyz"
curl -s -w ' HTTP %{http_code}\n' -X POST "$url/increment"
```

Ответы по порядку:

```text
{"status": "alive"} HTTP 200
{"error": "storage_unavailable"} HTTP 503
{"error": "storage_unavailable"} HTTP 503
```

Само приложение отвечало, но прибавить единицу без Redis не получилось. Затем запустил Redis и дождался ответа от `/readyz`:

```sh
docker compose start redis
curl -fsS --retry 12 --retry-all-errors --max-time 3 "$url/readyz"
curl "$url/"
curl -X POST "$url/increment"
```

`/readyz` вернул `"status": "ready"`. При чтении в счётчике по-прежнему было 3, а следующий POST вернул 4. Приложение для этого перезапускать не пришлось.
