# BDT (Brusnika Deployment Tool)
Инструмент для деплоя сервисов Брусники
- Генерирует деплой-конфиг для Nomad
- Создает политику для чтения своих секретов для приложения в Vault
- Создает конфиг сайта на mainproxy (cld-prx-01)
- Создает БД в кластере Postgres на сервисе [dbprime](https://gitlab.brusnika.tech/inf/dbprime)

## Переменные
**Полужирным** обязательные переменные

| Переменная | По-умолчанию | Комментарий |
|----------|---------|----------|
| `realm` | `staging` | Среда для деплоймента |
| `region` | `gld` | Регион расположения кластера |
| `datacenters` | `{`<br>    `"production": ["gld-1"],`<br>`  "staging": ["gld-stg-1"]`<br>`}` | Датацентры в которых расположены кластера worker-нод |
| `ports` | `{`<br>`  "api": { "to": 3000 },`<br>`  "http": { "to": 80 }`<br>`}` | Маппинг портов |
| `containers` | `{`<br>`  "backend": { "configs": [ "config.yaml" ] },`<br>`  "frontend": null`<br>`}` | Контейнеры которые будут запущены |
| `proxy` | `{}` | Конфигурация проксирования сервиса |
| **`proxy.host`** | `not defined`  | URL на котором будет опубликован сервис |
| `proxy.locations[]` | `{`<br>    `"container": "frontend",`<br>`  "name": "/"`<br>`}, {`<br>`   "container": "backend",`<br>`   "name": "/backend/"` <br>`}` | Проксирование location на определенный контейнер |
| `app_name` | `<service folder name>` | Название сервиса. Берется из имени папки, в которой лежит deploy.yaml |
| `image_version` | `latest` | Версия образов которые будут задеплоены |
| `snapshot_repo_url` | `snapshot-repo.brusnika.tech/brusnika/` | стейдж репозитория имеджей |
| `release_repo_url` | `release-repo.brusnika.tech/brusnika/` | продакшн репозитория имеджей |
| `repo_url` | `"{{ snapshot_repo_url }}"` | Зависит от текущего `realm` |
| `containers.<container-name>.image` | `"{{ repo_url }}{{ app_name }}-{{ container }}:{{ image_version }}"` | Имедж из которого будет запущен контейнер |
| `containers.<container-name>.resources` | `{}` | Управление ресурсами выделямыми на контейнер |
| `containers.<container-name>.command` | `` | Комманда запуска при старте контейнера |
| `containers.<container-name>.configs` | `[]` | Список файлов для монтирования в контейнер с хоста. По-умолчанию файлы монтируются в корень, если надо в другое место, то надо передать объект вида {'<имя_файла>': {'target': '<путь_монтирования/>'}}  |
| `resources.cpu` | `10` | Гарантированая производительность ЦПУ в МГц. Не используется одновременно с `resources.cores` |
| `resources.cores` | `null` | Гарантированное количество ядер ЦПУ. Не используется одновременно с `resources.cpu` |
| `resources.memory` | `50` | Гарантированный объем RAM в Мб |
| `resources.memory_max` | `100` | Максимальный объем RAM в Мб |

## Примеры конфига
Сервис у которого один контейнер backend с одним конфиг-файлом:
```yaml
---
- hosts: localhost
  gather_facts: no
  collections:
    - brusnika.deploy
  roles:
    - deploy
  vars:
    containers:
      backend:
        configs:
        - config.yaml
    proxy:
      host: operation-monitor.staging.brusnika.tech

```
Стандартный сервис + третий контейнер не из нашего репозитория:
```yaml
---
- hosts: localhost
  gather_facts: no
  collections:
    - brusnika.deploy
  roles:
    - deploy
  vars:
    containers:
      frontend:
      backend:
        configs:
        - config.yaml
      redis:
        image: "redis:7"
    proxy:
      host: pm-monitor.staging.brusnika.tech
```

Нестандартный высоконагруженный сервис:
```yaml
---
- hosts: localhost
  gather_facts: no
  collections:
    - brusnika.deploy
  roles:
    - deploy
  vars:
    containers:
      airflow:
        image: "bitnami/airflow"
        ports:
          http:
            to: 8080
        resources:
          cpu: 100
          memory: 256
          memory_max: 1024
    proxy:
      host: etl.brusnika.tech
      locations:
        - name: /
          container: airflow
```

Монтирование переменных окружения из файла
```yaml
...
  vars:
    containers:
      backend:
        configs:
        - env:
            myenvfile
...
```

Монтирование конфиг файла в директорию отличную от корня
```yaml
---
...
  vars:
    containers:
      backend:
        configs:
        - config.yaml:
            target: myconfig/
```

### Управление базами данных
```yaml
---
...
vars:
  rdbms:
    - host: general-staging
      port: 20901
      databases:
        - name: brusnika_navigator
          ext: ['uuid-ossp']
      users:
        - name: a.safin
          member: ['readonly']
        - name: m.sholomov
          member: ['brusnika_navigator']
```

## Зависимости
* community.hashi_vault

## Необходимые пакеты
* hvac
