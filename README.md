# GRE-туннель с ограничением полосы и мониторингом на Ansible

Ansible-плейбук разворачивает стенд из трёх VM на Debian 13:

- GRE-туннель между двумя iperf3-серверами;
- ограничение полосы туннеля до **10 Мбит/с в каждую сторону** через `tc`;
- firewall на узлах туннеля;
- мониторинг на Prometheus и Grafana в Docker Compose.

Плейбук идемпотентен: повторный запуск ничего не меняет.

## Содержание

- [Результат](#результат)
- [Архитектура](#архитектура)
- [Состав плейбука](#состав-плейбука)
- [Требования](#требования)
- [Быстрый старт](#быстрый-старт)
- [Переменные](#переменные)
- [Как это устроено](#как-это-устроено)
- [Проверка критериев приёмки](#проверка-критериев-приёмки)
- [Мониторинг](#мониторинг)
- [Эксплуатация](#эксплуатация)
- [Проверки кода](#проверки-кода)
- [Безопасность](#безопасность)
- [Ограничения](#ограничения)
- [Структура репозитория](#структура-репозитория)

---

## Результат

| Критерий задания | Результат на стенде |
|---|---|
| Скорость через туннель стабильно около 10 Мбит/с | **9.7–9.8 Мбит/с** в обе стороны |
| Прямое соединение показывает максимальную скорость сети | 1370–2166 Мбит/с, ограничение на него не действует |
| Дашборд в Grafana показывает актуальные метрики обеих VM | CPU, RAM, физические интерфейсы и туннель `gre1`, обновление каждые 5 с |
| На графике туннеля под нагрузкой виден потолок около 10 Мбит/с | панель TX `gre1` с линией лимита 10 Мбит/с |
| Плейбук идемпотентен | повторный прогон: `changed=0` на всех хостах |

Туннель, шейпер и firewall поднимаются при загрузке без участия Ansible.

---

## Архитектура

```
              управляющая сеть 192.168.56.0/24 (SSH, scrape :9100)
   ┌──────────────────────────────────────────────────────────────┐
   │                                                              │
   │  ┌────────────────────┐        GRE        ┌────────────────────┐
   │  │  iperf-server-01   │◄═════════════════►│  iperf-server-02   │
   │  │  192.168.56.11     │   10.20.20.0/30   │  192.168.56.12     │
   │  │  underlay          │  ≤ 10 Мбит/с      │  underlay          │
   │  │  192.168.100.11    │  в каждую сторону │  192.168.100.12    │
   │  │  gre1 10.20.20.1   │                   │  gre1 10.20.20.2   │
   │  │                    │                   │                    │
   │  │  iperf3 -s         │                   │  iperf3 -s         │
   │  │  nftables          │                   │  nftables          │
   │  │  node_exporter     │                   │  node_exporter     │
   │  └─────────▲──────────┘                   └─────────▲──────────┘
   │            │             scrape :9100               │
   │            └──────────────────┬─────────────────────┘
   │                     ┌─────────┴──────────┐
   │                     │   monitoring-01    │
   │                     │   192.168.56.13    │
   │                     │  Prometheus :9090  │
   │                     │  Grafana    :3000  │
   │                     │  (Docker Compose)  │
   │                     └────────────────────┘
   └──────────────────────────────────────────────────────────────┘
```

| Хост | Назначение | Управляющий адрес | Underlay туннеля | Адрес в туннеле |
|---|---|---|---|---|
| `iperf-server-01` | iperf3, firewall, node_exporter, узел туннеля | 192.168.56.11 | 192.168.100.11 | 10.20.20.1 |
| `iperf-server-02` | iperf3, firewall, node_exporter, узел туннеля | 192.168.56.12 | 192.168.100.12 | 10.20.20.2 |
| `monitoring-01` | Prometheus и Grafana в Docker Compose | 192.168.56.13 | — | — |

Туннель идёт по выделенной сети 192.168.100.0/24 (`enp0s8`). MTU `gre1` задаётся явно: 1476 байт = 1500 − 20 (внешний IP-заголовок) − 4 (заголовок GRE), поэтому фрагментации нет.

---

## Состав плейбука

`playbooks/site.yml` состоит из двух play.

**Play 1 — `iperf_servers`:**

| Роль | Что делает |
|---|---|
| `firewall` | nftables: политика `drop` на входе, разрешено только нужное (см. [Безопасность](#безопасность)) |
| `gre_tunnel` | интерфейс `gre1`, адресация 10.20.20.0/30, MTU 1476, systemd-юнит `gre-tunnel.service` для подъёма при загрузке |
| `traffic_control` | TBF на исходящем трафике `gre1`, привязанный к юниту туннеля через systemd drop-in |
| `iperf` | iperf3 в режиме сервера, systemd-юнит от непривилегированного пользователя |
| `node_exporter` | `prometheus-node-exporter`, порт 9100 |

**Play 2 — `monitoring`:**

| Роль | Что делает |
|---|---|
| `docker` | Docker CE и плагин Compose из официального репозитория Docker |
| `monitoring_stack` | Prometheus и Grafana в Compose, provisioning datasource и дашборда |

Источники ПО:

- Docker CE — официальный репозиторий Docker, канал `stable`, версии пакетов закреплены (`docker_version`, `docker_containerd_version`, `docker_compose_plugin_version` в `roles/docker/defaults/main.yml`);
- iperf3, prometheus-node-exporter, nftables — официальные репозитории Debian;
- образы `prom/prometheus` и `grafana/grafana` — официальные, версии закреплены.

---

## Требования

**Управляющая машина:**

- ansible-core 2.15 или новее (этого требует коллекция `community.docker` 4.x);
- Python 3.9+;
- SSH-доступ к хостам, пользователь с правом `sudo`;
- ключи хостов в `known_hosts`: в `ansible.cfg` включён `host_key_checking`.

**Целевые хосты:**

- Debian 13 (trixie);
- на iperf-серверах — два сетевых интерфейса: управляющий и `enp0s8` для туннеля;
- на `monitoring-01` — не менее 2 ГБ RAM и доступ к Docker Hub.

---

## Быстрый старт

**1. Клонировать репозиторий:**

```bash
git clone https://github.com/greenolik-hub/devopstest.git
```

**2. Установить коллекции:**

```bash
cd devopstest && ansible-galaxy collection install -r requirements.yml
```

**3. Описать хосты** в `inventory/production/hosts.yml`:

```yaml
all:
  children:
    iperf_servers:
      hosts:
        iperf-server-01:
          ansible_host: 192.168.56.11      # управляющий адрес
          ansible_user: user1
          gre_local_ip: 192.168.100.11     # underlay туннеля
          gre_remote_ip: 192.168.100.12
          gre_tunnel_ip: 10.20.20.1        # адрес внутри туннеля (/30)
        iperf-server-02:
          ansible_host: 192.168.56.12
          ansible_user: user2
          gre_local_ip: 192.168.100.12
          gre_remote_ip: 192.168.100.11
          gre_tunnel_ip: 10.20.20.2
    monitoring:
      hosts:
        monitoring-01:
          ansible_host: 192.168.56.13
          ansible_user: user3
```

**4. Проверить связность:**

```bash
ansible all -m ping --ask-vault-pass
```

**5. Развернуть стенд:**

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

**6. Проверить идемпотентность.** Повторный запуск должен дать `changed=0` на всех хостах:

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

### Секреты

Файлы `inventory/production/host_vars/*/vault.yml` зашифрованы Ansible Vault. Пароль передаётся через `--ask-vault-pass`. Чтобы не вводить его каждый раз, положите пароль в `.vault_pass` (файл в `.gitignore`) и раскомментируйте `vault_password_file` в `ansible.cfg`.

Пароль администратора Grafana задаётся переменной `vault_grafana_admin_password` в зашифрованном файле хоста мониторинга (не короче 12 символов):

```bash
ansible-vault edit inventory/production/host_vars/monitoring-01/vault.yml
```

Добавьте в файл строку `vault_grafana_admin_password: "<пароль>"`, остальные переменные оставьте как есть. Без пароля роль `monitoring_stack` останавливается на проверке, Grafana с `admin/admin` не разворачивается. Логин администратора — `grafana_admin_user` (по умолчанию `grafana_admin`), он не секретный и лежит в defaults роли.

Логин и пароль применяются только при **первой** инициализации базы Grafana. На уже развёрнутом стенде (где Grafana создала пользователя `admin`) переименуйте пользователя через API и задайте пароль. Выполняйте на `monitoring-01`:

```bash
curl -s -X PUT -u 'admin:<текущий пароль>' -H 'Content-Type: application/json' \
  -d '{"login":"grafana_admin"}' http://localhost:3000/api/users/1
```

```bash
sudo docker exec grafana grafana cli admin reset-admin-password '<пароль из Vault>'
```

Другой вариант: остановить стек и удалить `/opt/monitoring/grafana`. Дашборды и datasource создаются provisioning'ом, поэтому они не потеряются.

### Запуск по тегам

| Тег | Что выполняется |
|---|---|
| `firewall` | `firewall` |
| `tunnel` | `gre_tunnel` + `traffic_control` |
| `shaping` | `traffic_control` |
| `iperf` | `iperf` |
| `node_exporter` | `node_exporter` |
| `docker` | `docker` |
| `monitoring` | `node_exporter` на iperf-серверах + `docker` + `monitoring_stack` |

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass --tags shaping
```

Сухой прогон уже развёрнутого стенда:

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass --check --diff
```

На чистых хостах `--check` падает на установке Docker: в режиме проверки репозиторий Docker не добавляется, и пакет `docker-ce` не находится. Это ожидаемо.

---

## Переменные

Значения по умолчанию лежат в `roles/*/defaults/main.yml`. Параметры, которые используют несколько ролей или play, собраны в `inventory/production/group_vars/all.yml`. Каждый такой параметр записан в одном месте. В inventory задаётся только то, что отличается для конкретного стенда.

Переменные каждой роли описаны в её `meta/argument_specs.yml` — такой файл есть у всех семи ролей. Ansible проверяет их перед запуском роли и останавливается с понятной ошибкой, если обязательная переменная не задана или имеет неверный тип.

**Общие параметры стенда (`group_vars/all.yml`):**

| Переменная | Значение | Кто использует |
|---|---|---|
| `gre_interface` | `gre1` | `gre_tunnel`, `traffic_control`, дашборд Grafana |
| `traffic_control_rate` | `10mbit` | `traffic_control`, линия лимита на дашборде |
| `node_exporter_port` | `9100` | `firewall`, `node_exporter`, Prometheus |
| `iperf3_port` | `5201` | `iperf`, `firewall` |

Зависимости между ролями: `traffic_control` требует интерфейс и юнит от `gre_tunnel`; `firewall` использует `gre_remote_ip`, `iperf3_port`, `node_exporter_port` и группу `monitoring`; `monitoring_stack` требует Docker из роли `docker` и общие параметры выше. Порядок ролей в `site.yml` соответствует этим зависимостям.

**Задаются в inventory для каждого iperf-сервера:**

| Переменная | Назначение |
|---|---|
| `gre_local_ip` | свой адрес в underlay-сети |
| `gre_remote_ip` | адрес второго конца туннеля в underlay-сети |
| `gre_tunnel_ip` | адрес внутри туннеля |

**Значения по умолчанию:**

| Переменная | Роль | Значение |
|---|---|---|
| `gre_prefix` | `gre_tunnel` | `30` |
| `gre_mtu` | `gre_tunnel` | `1476` |
| `traffic_control_burst` | `traffic_control` | `64kb` |
| `traffic_control_latency` | `traffic_control` | `400ms` |
| `prometheus_image` | `monitoring_stack` | `prom/prometheus:v3.16.0` |
| `grafana_image` | `monitoring_stack` | `grafana/grafana:13.3.0` |
| `prometheus_node_exporter_group` | `monitoring_stack` | `iperf_servers` |
| `monitoring_data_dir` | `monitoring_stack` | `/opt/monitoring` |
| `prometheus_bind_address` | `monitoring_stack` | `127.0.0.1` |
| `docker_version` | `docker` | `29.9.3` (также `docker_containerd_version`, `docker_compose_plugin_version`) |
| `grafana_bind_address` | `monitoring_stack` | `0.0.0.0` (для доступа только через SSH-туннель задайте `127.0.0.1`) |
| `grafana_admin_user` | `monitoring_stack` | `grafana_admin` |

---

## Как это устроено

### Туннель

Скрипт `/usr/local/sbin/configure-gre.sh` создаёт `gre1`, а если интерфейс уже есть — обновляет его параметры (`ip tunnel change`). Затем назначает адрес (`ip address replace`) и поднимает интерфейс с MTU 1476. Скрипт запускает юнит `gre-tunnel.service` (`Type=oneshot`), поэтому туннель поднимается при загрузке. При остановке юнита интерфейс удаляется.

### Ограничение полосы

На исходящий трафик `gre1` каждой VM ставится TBF (Token Bucket Filter):

```
iperf-server-01: tc ... dev gre1  →  направление 01 → 02
iperf-server-02: tc ... dev gre1  →  направление 02 → 01
```

`tc` управляет исходящей очередью, поэтому каждое направление ограничивается на своём отправителе. Ограничивать входящий трафик через `ifb` не нужно. Физический интерфейс не затронут: прямое соединение работает на полной скорости.

**Почему shaping, а не policing.** Policing (`tc filter ... action police`) отбрасывает пакеты сверх лимита. TCP отвечает на потери уменьшением окна, и скорость скачет пилой, с ретрансмиссиями. Shaping ставит лишние пакеты в очередь и выпускает их равномерно: потерь нет, скорость ровно держится у лимита. Это и требует критерий приёмки — «скорость стабильно держится около 10 Мбит/с».

**Почему TBF.** Одна дисциплина и одна команда `tc`: её просто описать в Ansible и просто проверить. Для одного ограничения на интерфейс дерево классов HTB избыточно.

**Почему `burst 64kb`.** TBF выдаёт порцию данных при каждом срабатывании таймера. Нужная частота срабатываний равна `rate / burst`:

| burst | нужно срабатываний в секунду при 10 Мбит/с |
|---|---|
| 8 КБ (типовое значение из примеров) | 153 |
| 64 КБ | 19 |

На виртуальных машинах таймер работает с задержками, поэтому маленький `burst` занижает фактическую скорость. 64 КБ дают многократный запас. Есть и второй эффект: GSO-пакет может быть размером до 64 КБ, и при меньшем `burst` TBF режет его программно. Цена большого `burst` — допустимый всплеск до 64 КБ, около 50 мс трафика. На секундном окне iperf3 и минутном окне Grafana его не видно.

**Как шейпер переживает перезагрузку.** Настройки `tc` живут в памяти ядра и исчезают вместе с интерфейсом. Поэтому TBF применяется не задачей Ansible, а drop-in'ом к юниту туннеля:

```ini
# /etc/systemd/system/gre-tunnel.service.d/10-tbf.conf
[Service]
ExecStartPost=/usr/sbin/tc qdisc replace dev gre1 root tbf rate 10mbit burst 64kb latency 400ms
```

Шейпер ставится при каждом старте `gre-tunnel.service`: при загрузке и при рестарте туннеля. Ansible меняет drop-in только при изменении переменных, и тогда handler сразу применяет новые параметры. Если TBF сняли вручную, следующий прогон плейбука его восстановит.

### Firewall

nftables на iperf-серверах: политика `drop` на входе, разрешения перечислены в разделе [Безопасность](#безопасность). Перед записью конфиг проверяется командой `nft -c`, поэтому конфиг с ошибкой не применится. Правила загружает штатный сервис `nftables`, поэтому они действуют и после перезагрузки.

### Мониторинг

- Prometheus опрашивает `node_exporter` на iperf-серверах каждые 5 секунд. Список таргетов генерируется из группы `iperf_servers` в inventory, к каждой серии добавляется метка `host` с именем хоста.
- Datasource Prometheus и дашборд создаются в Grafana через provisioning, ручных шагов нет. UID datasource фиксирован (`prometheus`), дашборд ссылается на него.
- Дашборд — шаблон: имя туннельного интерфейса и линия лимита подставляются из `gre_interface` и `traffic_control_rate`, поэтому при смене лимита дашборд остаётся верным. Для лимита поддерживается формат `<число>mbit`.
- Конфиги монтируются в контейнеры каталогами. При изменении конфига handler перезапускает только нужный сервис.
- Данные Prometheus и Grafana хранятся на хосте в `/opt/monitoring` и переживают пересоздание контейнеров.

### Идемпотентность

- файлы раскладываются модулями `template` и `copy`, которые сравнивают содержимое;
- пакеты ставятся через `apt` с `state: present`, сервисы приводятся к состоянию `started` + `enabled`;
- контейнерами управляет `community.docker.docker_compose_v2`, он сам сверяет состояние;
- команды, которые только читают состояние, помечены `changed_when: false`;
- handlers срабатывают только при реальных изменениях.

- команда `tc` описана один раз (`roles/traffic_control/vars/main.yml`): drop-in, handler и проверка дрейфа используют её. Если TBF сняли вручную, проверка уведомляет тот же handler, поэтому при первом прогоне TBF применяется один раз;

Качество кода: `ansible-lint` в профиле `production` и `yamllint` проходят без замечаний.

---

## Проверка критериев приёмки

### Туннель

```bash
ip -d link show gre1
```

```bash
ping -c 3 10.20.20.2        # с iperf-server-01
```

Ожидается MTU 1476 и успешный пинг.

### Шейпер

На обеих VM:

```bash
tc -s qdisc show dev gre1
```

Ожидаемый вывод:

```
qdisc tbf 8001: root refcnt 2 rate 10Mbit burst 64Kb lat 400ms
 Sent ... (dropped 0, overlimits ... requeues 0)
```

`dropped 0` подтверждает, что работает shaping: лишние пакеты ждут в очереди, а не отбрасываются.

Проверка после перезагрузки: выполните `sudo reboot`, затем снова `tc qdisc show dev gre1`. TBF должен быть на месте без запуска Ansible.

### Скорость через туннель

iperf3-сервер обслуживает один тест за раз, поэтому тесты запускаются по очереди.

```bash
# с iperf-server-01: направление 01 → 02
iperf3 -c 10.20.20.2 -t 60 -i 5 -f m
```

```bash
# с iperf-server-02: направление 02 → 01
iperf3 -c 10.20.20.1 -t 60 -i 5 -f m
```

Флаг `-R` меняет направление передачи, а не адрес: с ним каждое направление можно проверить с любой стороны.

Результат на стенде: **9.7–9.8 Мбит/с в обе стороны**, скорость ровная на всём интервале.

### Прямое соединение

```bash
iperf3 -c 192.168.100.12 -t 10 -f m    # с iperf-server-01
```

Результат на стенде: 1370–2166 Мбит/с. Ограничение действует только на туннель.

---

## Мониторинг

| Сервис | Адрес | Доступ |
|---|---|---|
| Grafana | http://192.168.56.13:3000/d/node-overview | `grafana_admin` / `vault_grafana_admin_password` |
| Prometheus | `127.0.0.1:9090` на `monitoring-01` | только локально, без авторизации |

Prometheus слушает только `127.0.0.1`, Grafana обращается к нему по внутренней сети Compose. Чтобы открыть его интерфейс с рабочей станции, используйте SSH-туннель и откройте http://localhost:9090:

```bash
ssh -L 9090:127.0.0.1:9090 user3@192.168.56.13
```

Дашборд **Node Overview — GRE stand** (папка *Infrastructure*) показывает обе VM одновременно, каждая линия подписана именем хоста:

| Панель | Что показывает |
|---|---|
| CPU Usage | загрузка CPU |
| RAM Usage | использование памяти |
| Main interfaces traffic | RX/TX физических интерфейсов |
| Tunnel gre1 TX | исходящий трафик туннеля — его ограничивает TBF. Красная линия отмечает 10 Мбит/с |
| Tunnel gre1 RX | входящий трафик туннеля |

Переменные `host` и `iface` в шапке дашборда по умолчанию стоят в положении *All*. Ими можно сузить вид до одной VM или одного интерфейса.

Чтобы увидеть потолок, дайте нагрузку и смотрите панель **Tunnel gre1 TX**:

```bash
iperf3 -c 10.20.20.2 -t 120 -f m
```

Панель использует запрос с минутным окном, которое сглаживает кратковременные колебания:

```promql
rate(node_network_transmit_bytes_total{device="gre1"}[1m]) * 8
```

---

## Эксплуатация

### Изменение параметров ограничения

Измените `traffic_control_rate`, `traffic_control_burst` или `traffic_control_latency` (в defaults роли или переопределите в inventory) и примените. Туннель при этом не перезапускается:

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass --tags shaping
```

### Добавление хоста в мониторинг

Добавьте хост в группу `iperf_servers` и запустите плейбук. Prometheus начнёт его опрашивать, а на дашборде он появится автоматически.

### Диагностика стека мониторинга

На `monitoring-01` — состояние контейнеров и Grafana:

```bash
cd /opt/monitoring && sudo docker compose ps && curl -s http://localhost:3000/api/health
```

Состояние таргетов Prometheus:

```bash
curl -s http://localhost:9090/api/v1/targets | grep -oE '"scrapeUrl":"[^"]+"|"health":"[a-z]+"'
```

Логи сервиса:

```bash
sudo docker compose logs --tail=50 grafana
```

Если Grafana не стартует с ошибкой `Datasource provisioning error: data source not found`, в её базе есть datasource `Prometheus` с другим UID (например, созданный вручную). Удалите его в интерфейсе Grafana или удалите каталог данных `/opt/monitoring/grafana` и запустите плейбук заново.

---

## Проверки кода

Конфигурация линтеров лежит в `.yamllint` и `.ansible-lint` (профиль `production`). Установка:

```bash
pip install ansible-core==2.15.13 ansible-lint==6.22.2 yamllint==1.37.1
ansible-galaxy collection install -r requirements.yml
```

Запуск из корня репозитория:

```bash
yamllint -s .
```

```bash
ansible-lint
```

```bash
ansible-playbook playbooks/site.yml --syntax-check
```

Файлы `host_vars/*/vault.yml` зашифрованы. Без пароля `ansible-lint` пропускает их с предупреждением, а `yamllint` исключает их в `.yamllint`. Чтобы проверить и их, передайте пароль через переменную окружения (у `ansible-lint` нет флага для пароля vault):

```bash
ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass ansible-lint
```

Те же проверки запускает GitHub Actions (`.github/workflows/lint.yml`) на каждый `push` и `pull_request`. В CI зашифрованные `vault.yml` заменяются фиктивными файлами, реальные секреты не используются. Версии инструментов в workflow закреплены: обновляйте их осознанно.

Линтеры и `--syntax-check` не заменяют проверку на серверах: идемпотентность, поведение после перезагрузки и скорость через туннель проверяются по разделу [Проверка критериев приёмки](#проверка-критериев-приёмки).

---

## Безопасность

- Секреты хранятся только в зашифрованных файлах Ansible Vault.
- iperf3 работает от динамического непривилегированного пользователя с ограничениями systemd (`DynamicUser`, `NoNewPrivileges`, `PrivateDevices`, `Protect*`).
- Регистрация пользователей в Grafana отключена, пароль администратора обязателен и задаётся через Vault: без него плейбук останавливается.
- Prometheus опубликован только на `127.0.0.1` хоста мониторинга.
- На iperf-серверах работает nftables, входящий трафик по умолчанию запрещён:

| Разрешено | Откуда |
|---|---|
| установленные соединения, `lo`, ICMP/ICMPv6 | отовсюду |
| SSH (`ansible_port`, по умолчанию 22) | отовсюду |
| GRE (IP protocol 47) | только underlay-адрес второго конца туннеля |
| iperf3 5201/tcp и 5201/udp | отовсюду (тест через туннель и напрямую) |
| node_exporter 9100/tcp | только хосты группы `monitoring` |

Firewall скорость не ограничивает. Посмотреть действующие правила:

```bash
sudo nft list ruleset
```

---

## Ограничения

- **Firewall только на узлах туннеля.** Так требует схема задания. На `monitoring-01` порт Grafana 3000 открыт в LAN, Prometheus доступен только локально. Роль `firewall` на этот хост применять нельзя: её `flush ruleset` удалит правила Docker и сломает сеть контейнеров. При размещении в общей сети закройте порт 3000 на уровне сети или задайте `grafana_bind_address: 127.0.0.1` и ходите в Grafana через SSH-туннель. Трафик к Grafana идёт по HTTP, логин и пароль не шифруются. Формат `traffic_control_rate` проверяет роль `traffic_control` до изменений на хосте.
- **Ручное пересоздание `gre1`.** Если пересоздать интерфейс через `ip link` в обход юнита, TBF пропадёт. Верните его командой `sudo systemctl restart gre-tunnel` или прогоном с тегом `shaping`.
- **Ручное изменение TBF.** Если поменять параметры TBF вручную через `tc`, Ansible этого не заметит: он проверяет только наличие TBF, а параметры берёт из drop-in.
- **Задержка под нагрузкой.** `latency 400ms` при 10 Мбит/с означает очередь около 500 КБ: под полной нагрузкой задержка в туннеле может вырасти до 400 мс. Для замеров пропускной способности это не мешает. Для интерактивного трафика стоит уменьшить `traffic_control_latency` до 50–100 мс.
- **Повышение версий.** Образы Prometheus и Grafana закреплены. Обновление — это осознанная смена версии в `roles/monitoring_stack/defaults/main.yml`.

---

## Структура репозитория

```
.
├── ansible.cfg                   # inventory, roles_path, SSH, become
├── requirements.yml              # community.docker (версия закреплена)
├── .ansible-lint / .yamllint     # конфигурация линтеров
├── .github/workflows/lint.yml    # CI: yamllint, ansible-lint, syntax-check
├── inventory/production/
│   ├── hosts.yml                 # хосты, адреса, параметры туннеля
│   ├── group_vars/all.yml        # общие параметры стенда (одно место на параметр)
│   └── host_vars/*/vault.yml     # секреты (Ansible Vault)
├── playbooks/
│   └── site.yml                  # точка входа
└── roles/
    ├── firewall/                 # nftables на iperf-серверах
    ├── gre_tunnel/               # gre1 и systemd-юнит
    ├── traffic_control/          # TBF через systemd drop-in
    ├── iperf/                    # iperf3-сервер
    ├── node_exporter/            # экспортер метрик
    ├── docker/                   # Docker CE и Compose
    └── monitoring_stack/         # Prometheus, Grafana (Compose)
```
