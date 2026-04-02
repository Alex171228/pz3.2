# Практическое задание 3
## Шишков А.Д. ЭФМО-02-25
## Тема
Структурированное логирование в серверных приложениях.

## Цель
Научиться внедрять структурированные логи в сервис и применять единый стандарт логирования для диагностики и эксплуатации.

---

## 1. Выбор логгера

Выбран **zap** (`go.uber.org/zap`).

Причины:

- Выдаёт JSON по умолчанию — логи сразу готовы для парсинга (ELK, Loki, jq).
- Высокая производительность — zero-allocation в hot path, что важно для микросервисов под нагрузкой.
- Типизированные поля (`zap.String`, `zap.Int`, `zap.Error`) исключают случайную запись чувствительных данных через `fmt.Sprintf`.
- Широко используется в production Go-проектах.

---

## 2. Стандарт полей логов

Во всех сервисах (auth, tasks) используется единая схема полей:

### Обязательные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `level` | string | Уровень: debug / info / warn / error |
| `ts` | string | Время события (ISO 8601) |
| `msg` | string | Описание события |
| `service` | string | Имя сервиса: `auth` или `tasks` |
| `request_id` | string | X-Request-ID (из заголовка или сгенерированный UUID) |

### Поля access log (middleware)

| Поле | Тип | Описание |
|------|-----|----------|
| `method` | string | HTTP-метод (GET, POST, PATCH, DELETE) |
| `path` | string | Путь запроса (`/v1/tasks`) |
| `status` | int | Код ответа (200, 401, 503...) |
| `duration_ms` | float | Длительность обработки в миллисекундах |
| `remote_ip` | string | IP-адрес клиента |

### Поля для ошибок и контекста

| Поле | Тип | Описание |
|------|-----|----------|
| `error` | string | Текст ошибки (без секретов) |
| `component` | string | Компонент: `handler`, `auth_middleware`, `auth_client_http`, `auth_client_grpc` |
| `task_id` | string | ID задачи (при CRUD-операциях) |
| `subject` | string | Субъект из токена (при успешной верификации) |
| `has_auth` | bool | Факт наличия токена (без самого значения) |

### Что запрещено логировать

- Пароли
- Токены доступа и refresh-токены
- Секреты и ключи
- Содержимое cookies

---

## 3. Примеры лог-событий

### 3.1. Успешный запрос — создание задачи


```bash
curl -i -X POST http://<SERVER_IP>:8082/v1/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer demo-token" \
  -H "X-Request-ID: pz19-002" \
  -d '{"title":"Logs","description":"Implement zap","due_date":"2026-01-12"}'
```

201 Created. В логах Tasks видно цепочку: calling auth HTTP verify → auth HTTP verify: success → token verified → task created → request completed. В логах Auth — token verified → request completed (200).

![Успешный запрос — создание задачи](docs/images/3_1_success.png)


### 3.2. Запрос с ошибкой — невалидный токен

```bash
curl -i http://<SERVER_IP>:8082/v1/tasks \
  -H "Authorization: Bearer invalid-token" \
  -H "X-Request-ID: pz19-003"
```

401 Unauthorized. В логах Tasks уровень `warn` — `auth HTTP verify: unauthorized`, `invalid token`. Токен не попадает в логи (только `has_auth: true`). В логах Auth — `token verification failed` (уровень `warn`).


![Запрос с невалидным токеном](docs/images/3_2_error.png)


### 3.3. Межсервисный вызов — корреляция по request-id

Шаг 1 — получить токен:

```bash
curl -s -X POST http://<SERVER_IP>:8081/v1/auth/login \
  -H "Content-Type: application/json" \
  -H "X-Request-ID: pz19-010" \
  -d '{"username":"student","password":"student"}'
```

Шаг 2 — создать задачу (request-id видно в обоих сервисах):

```bash
curl -i -X POST http://<SERVER_IP>:8082/v1/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer demo-token" \
  -H "X-Request-ID: pz19-011" \
  -d '{"title":"Cross-service","description":"Test correlation","due_date":"2026-01-12"}'
```

По `request_id: "pz19-011"` можно найти связанные события в логах обоих сервисов: Auth (`service: auth`, verify) и Tasks (`service: tasks`, task created).

![Корреляция request-id](docs/images/3_3_correlation.png)


---

## 4. Инструкция запуска и проверки

### Установка

```bash
git clone https://github.com/Alex171228/pz3.2.git
cd pz3.2
go mod download
```

### Запуск (2 терминала)

**Терминал 1 — Auth Service:**

```bash
export AUTH_PORT=8081
export AUTH_GRPC_PORT=50051
go run ./services/auth/cmd/auth
```

**Терминал 2 — Tasks Service:**

```bash
export TASKS_PORT=8082
go run ./services/tasks/cmd/tasks
```

### Проверка (3-й терминал)

Выполните запросы из пункта 3 или воспользуйтесь postman коллекцией https://github.com/Alex171228/pz3.2/blob/main/pz3_postman_collection.json

