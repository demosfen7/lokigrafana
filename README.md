# lokigrafana — стек логирования (Loki + Grafana)

Учебный стек для сбора и просмотра логов: Loki принимает логи по HTTP push,
Grafana их показывает. Служит общей инфраструктурой для других учебных
проектов на том же поддомене — например, [grafana](https://github.com/demosfen7/grafana)
(погодное приложение) шлёт в него свои логи напрямую.

## Ссылки

| Что | Где |
|---|---|
| Grafana (UI) | https://vibecoding.aithinglab.com/lokigrafana/ |
| Этот репозиторий | https://github.com/demosfen7/lokigrafana |
| Приложение, чьи логи сюда льются | https://github.com/demosfen7/grafana |

**Вход в Grafana:** изначально `admin` / `admin` (задано в `docker-compose.yml`), но при первом
входе через веб-интерфейс Grafana обычно просит сменить пароль — если `admin`/`admin` не подходит,
пароль уже сменён через UI при более раннем заходе.

---

## Из чего состоит

Всё в одном [`docker-compose.yml`](docker-compose.yml):

### `loki`

- `grafana/loki:2.9.6`, конфиг по умолчанию из образа (`local-config.yaml`), однобинарный режим.
- Данные — в именованном volume `loki-data`.
- Наружу (в интернет) не смотрит вообще — ни портов, ни Caddy-маршрута. Слушает `3100` только
  внутри docker-сетей.
- Состоит в двух сетях:
  - `default` (общая для этого compose-проекта) — по ней Grafana ходит в Loki по имени `loki`;
  - `loki-net` (внешняя, `external: true`) — через неё логи присылают **другие** compose-проекты,
    у которых свой собственный `default`. Сеть создаётся один раз вручную на сервере:
    ```
    docker network create loki-net
    ```
    (уже создана; нужно только если поднимаешь стек с нуля на новом сервере).

### `grafana`

- `grafana/grafana:10.4.2`.
- Логин/пароль и отключённая регистрация — прямо в `environment` (это учебный стенд, не продакшен).
- Висит за Caddy на суб-пути `/lokigrafana/`, поэтому `GF_SERVER_SERVE_FROM_SUB_PATH=true` +
  `GF_SERVER_ROOT_URL` с этим путём. **Важно:** для суб-пути в Caddy используется `handle`,
  а не `handle_path` — Grafana сама ожидает префикс `/lokigrafana` в пришедшем запросе и не работает,
  если Caddy его обрезает (проверено на практике — с `handle_path` был бесконечный редирект).
- Датасорс Loki прописан не руками, а через provisioning:
  [`provisioning/datasources/loki.yml`](provisioning/datasources/loki.yml) — url `http://loki:3100`,
  по имени сервиса в общей сети `default`.
- Данные (дашборды, пользователи) — в volume `grafana-data`, переживают пересоздание контейнера.

---

## Как что-то ещё подключить к Loki

Любой другой compose-проект на этом же сервере может слать логи в Loki:

1. Подключить его сервис к внешней сети `loki-net` (`networks: [loki-net]` + top-level
   `networks: { loki-net: { external: true } }`).
2. Пушить логи HTTP POST на `http://loki:3100/loki/api/v1/push` (формат — Loki push API).
   Пример реализации на Python: [`loki_logging.py`](https://github.com/demosfen7/grafana/blob/master/loki_logging.py)
   в проекте `grafana`.

Почему не Promtail: на этом сервере rootless Docker, и файлы JSON-логов контейнеров
(`~/.local/share/docker/containers/.../*.log`) на хосте принадлежат реальному пользователю
демона — контейнер с другим (remap-нутым) UID не может их прочитать через bind-mount.
Прямой push из приложения в Loki API эту проблему обходит.

---

## Сервер: пути и доступ

| Что | Значение |
|---|---|
| SSH | по алиасу из `~/.ssh/config`, пользователь rootless Docker, свой демон, без sudo |
| Папка деплоя | `/opt/vibecoding.aithinglab.com/lokigrafana/` |
| Порт наружу (Grafana) | `8093`, проброшен как `172.28.0.1:8093`, разрешён в `ufw` только из сети Caddy |
| Docker-сети | `default` (Grafana↔Loki) + `loki-net` (внешняя, общая для логов от других проектов) |

Общий Caddy-роутинг для всего поддомена — на сервере в
`/opt/briefgenerator/deploy/conf.d/vibecoding.aithinglab.com.caddy` (правится только вручную,
CI его не трогает). Копия для справки — [`deploy/vibecoding.aithinglab.com.caddy`](deploy/vibecoding.aithinglab.com.caddy).

---

## CI/CD

[`.github/workflows/deploy.yml`](.github/workflows/deploy.yml): при пуше в `master` копирует
`docker-compose.yml` и `provisioning/` на сервер, делает `docker compose pull && up -d`.
Образы официальные (grafana/loki, grafana/grafana) — свою сборку CI не делает.

Секреты репозитория (Settings → Secrets → Actions):

| Секрет | Значение |
|---|---|
| `SSH_HOST` | `<хост сервера>` |
| `SSH_USER` | `<пользователь rootless Docker>` |
| `SSH_PORT` | `<SSH-порт>` |
| `SSH_KEY` | `<содержимое приватного ключа этого пользователя>` |

---

## Структура проекта

```
lokigrafana/
├── docker-compose.yml                     # оба сервиса: loki + grafana
├── provisioning/datasources/loki.yml      # автонастройка датасорса Loki в Grafana
├── deploy/vibecoding.aithinglab.com.caddy # копия общего Caddy-роутинга — для справки
└── .github/workflows/deploy.yml           # деплой по SSH (без сборки образа)
```
