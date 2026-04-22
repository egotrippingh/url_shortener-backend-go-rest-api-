# URL Shortener

Сервис сокращения ссылок на Go. Приложение сохраняет исходный URL в SQLite, возвращает короткий алиас и по запросу `/{alias}` делает HTTP-редирект на оригинальный адрес.

## Возможности

- создание коротких ссылок через HTTP API;
- поддержка пользовательского алиаса;
- автоматическая генерация алиаса длиной 6 символов, если он не передан;
- хранение данных в SQLite;
- Basic Auth для создания коротких ссылок;
- структурированное логирование через `slog`;
- graceful shutdown при остановке процесса.

## Стек

- Go 1.20;
- `chi` для HTTP-роутинга;
- `cleanenv` для загрузки конфигурации;
- `sqlite3` для хранения ссылок;
- `slog` для логирования.

## Как это работает

1. Клиент отправляет `POST /url` с исходным URL и, при необходимости, своим алиасом.
2. Сервис сохраняет пару `alias -> url` в SQLite.
3. Если алиас не указан, он генерируется автоматически.
4. При запросе `GET /{alias}` сервис ищет URL по алиасу и возвращает редирект.

## API

### `POST /url`

Создает короткую ссылку.

Доступ:
- защищен Basic Auth;
- логин берется из `http_server.user`;
- пароль можно передать через `HTTP_SERVER_PASSWORD`.

Тело запроса:

```json
{
  "url": "https://example.com/articles/go",
  "alias": "go-article"
}
```

Поля:

- `url` - обязательное поле, должен быть валидный URL;
- `alias` - необязательное поле, если не передано, сервис сгенерирует значение автоматически.

Пример запроса:

```bash
curl -X POST http://localhost:8082/url \
  -u admin:secret \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/articles/go","alias":"go-article"}'
```

Успешный ответ:

```json
{
  "status": "OK",
  "alias": "go-article"
}
```

Если `alias` не передан:

```json
{
  "status": "OK",
  "alias": "aB93xQ"
}
```

Ошибка валидации:

```json
{
  "status": "Error",
  "error": "field URL is not a valid URL"
}
```

Ошибка при попытке сохранить уже существующий алиас:

```json
{
  "status": "Error",
  "error": "url already exists"
}
```

Примечание:
- в текущей реализации ошибки этого эндпоинта возвращаются в JSON через поле `status`, при этом HTTP-статус остается `200 OK`.

### `GET /{alias}`

Возвращает редирект на сохраненный URL.

Пример запроса:

```bash
curl -i http://localhost:8082/go-article
```

Успешный ответ:

- HTTP-статус: `302 Found`;
- заголовок `Location` содержит исходный URL.

Пример:

```http
HTTP/1.1 302 Found
Location: https://example.com/articles/go
```

Если алиас не найден, сервис возвращает JSON-ошибку:

```json
{
  "status": "Error",
  "error": "not found"
}
```

Примечание:
- как и в `POST /url`, в текущей реализации ошибка возвращается с HTTP-статусом `200 OK`.

## Конфигурация

Сервис читает конфигурацию из файла, путь к которому передается через переменную окружения `CONFIG_PATH`.

Обязательные параметры:

- `CONFIG_PATH` - путь до YAML-конфига;
- `storage_path` - путь до SQLite-базы;
- `http_server.user` - логин для Basic Auth;
- `HTTP_SERVER_PASSWORD` - пароль для Basic Auth.

Поддерживаемые параметры конфига:

- `env` - окружение (`local`, `dev`, `prod`), влияет на формат логов;
- `storage_path` - путь до файла SQLite;
- `http_server.address` - адрес HTTP-сервера;
- `http_server.timeout` - таймаут чтения и записи;
- `http_server.idle_timeout` - idle timeout;
- `http_server.user` - логин для Basic Auth;
- `http_server.password` - поле пароля в структуре конфига, но в текущем репозитории продовый сценарий предполагает передачу пароля через `HTTP_SERVER_PASSWORD`.

Пример конфига:

```yaml
env: "prod"
storage_path: "./storage.db"
http_server:
  address: "0.0.0.0:8082"
  timeout: 4s
  idle_timeout: 30s
  user: "admin"
```

Пароль удобнее передавать через переменную окружения:

```bash
export HTTP_SERVER_PASSWORD=secret
```

Готовый пример прод-конфига находится в [config/prod.yaml](config/prod.yaml).

## Локальный запуск

Требования:

- Go 1.20+;
- C toolchain для `github.com/mattn/go-sqlite3` (`gcc`, `build-essential` и т.д.).

1. Установить зависимости:

```bash
go mod download
```

2. Указать путь до конфига и пароль:

```bash
export CONFIG_PATH=./config/prod.yaml
export HTTP_SERVER_PASSWORD=secret
```

3. Запустить сервис:

```bash
go run ./cmd/url-shortener
```

После старта сервис будет слушать адрес из `http_server.address`.

## Сборка

```bash
go build -o bin/url-shortener ./cmd/url-shortener
```

## Тесты

Юнит-тесты:

```bash
go test ./internal/...
```

В репозитории также есть end-to-end тесты в [tests/url_shortener_test.go](tests/url_shortener_test.go). Они ожидают, что сервис уже запущен на `localhost:8082` и доступен с Basic Auth.

Запуск e2e-тестов:

```bash
go test ./tests/...
```

## Деплой

В репозитории есть пример unit-файла systemd: [deployment/url-shortener.service](deployment/url-shortener.service).

Текущий шаблон предполагает:

- бинарник в `/root/apps/url-shortener/url-shortener`;
- рабочую директорию `/root/apps/url-shortener`;
- переменные окружения в `/root/apps/url-shortener/config.env`.

## Структура проекта

```text
cmd/url-shortener/              точка входа приложения
config/                         YAML-конфиги
deployment/                     systemd unit file
internal/config/                загрузка конфигурации
internal/http-server/handlers/  HTTP-обработчики
internal/storage/sqlite/        SQLite-реализация хранилища
tests/                          end-to-end тесты
```
