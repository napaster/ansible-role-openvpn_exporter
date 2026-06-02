# openvpn_exporter

Раскатывает [natrontech/openvpn-exporter](https://github.com/natrontech/openvpn-exporter)
(Go binary из github releases) + systemd unit на хостах, где локально работает
`openvpn-server@<instance>` И есть локальный Prometheus, который скрейпит
`127.0.0.1:9176`.

Экспортер читает **OpenVPN status file** (без management TCP-socket'а), отдаёт
стандартные метрики `openvpn_up`, `openvpn_server_connected_clients`,
`openvpn_server_client_{sent,received}_bytes_total`,
`openvpn_status_update_time_seconds` и т.д. — те же, что и наш
[`prometheus_textfile_openvpn`](https://github.com/napaster/ansible-role-prometheus_textfile_openvpn),
чтобы дашборды/алерты были общие.

## Платформы

* **Arch Linux only** — роль `assert`'ом проверяет дистрибутив. Бинарь
  скачивается из github releases (linux_amd64 tarball). На Debian/EL пока
  не работает (нужны pacman-only пути и Arch-style systemd init).

## Требования

* `openvpn-server@<port>.service` запущен и пишет status в `/run/openvpn-server/<port>.status`
  (директива `status /run/openvpn-server/<port>.status <interval>` + `status-version 2`
  в `/etc/openvpn/server/<port>.conf`)
* Локальный Prometheus с scrape config на `127.0.0.1:9176`
* Сеть наружу для `get_url` (github releases CDN)

## Переменные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `openvpn_exporter_version` | `v1.2.2` | Pinned upstream version. Бампать вручную |
| `openvpn_exporter_bin` | `/usr/local/bin/openvpn-exporter` | Куда install бинарь |
| `openvpn_exporter_listen` | `127.0.0.1:9176` | Listen address. Менять на LAN/VPN только если нужен cross-host scrape |
| `openvpn_exporter_status_paths` | `[/run/openvpn-server/1191.status]` | Список status файлов для парсинга. **Override per-host** если порт ≠ 1191 (см. ниже) |
| `openvpn_exporter_ignore_individuals` | `false` | `--ignore.individuals` — скрыть per-CN метрики (только агрегаты) |

## Пример использования

`host_vars/<host>/openvpn_exporter.yml` (когда `openvpn-server@<port>` ≠ 1191):
```yaml
---
openvpn_exporter_status_paths:
  - "/run/openvpn-server/1192.status"
```

Playbook (см. [napaster/playbook `monitoring/deploy_openvpn_exporter.yml`](https://github.com/napaster/playbook/blob/master/monitoring/deploy_openvpn_exporter.yml)):
```yaml
- name: Deploy openvpn-exporter on collectors that run OpenVPN
  hosts: "{{ target | default('none') }}"
  become: true
  roles:
    - openvpn_exporter
```

Запуск:
```
ansible-playbook playbook/monitoring/deploy_openvpn_exporter.yml -e target=gw.ventcomplex.ru
```

Scrape config на хосте (в `host_vars/<host>/prometheus.yml`):
```yaml
prometheus_scrape_configs_extra:
  - job_name: 'openvpn'
    static_configs:
      - targets: ['127.0.0.1:9176']
        labels:
          instance: "{{ inventory_hostname }}"
          site: "{{ prometheus_site }}"
          cluster: "{{ prometheus_cluster }}"
```

## Альтернатива для VMs без локального Prometheus

Если на хосте только `node_exporter` (нет своего Prometheus, метрики тянет
центральная промка через `remote_write`) — используй
[`prometheus_textfile_openvpn`](https://github.com/napaster/ansible-role-prometheus_textfile_openvpn).
Там парсер на Python через `textfile_collector` node_exporter'а, не нужен отдельный
listener на :9176 и проброс портов наружу.

## Handlers

* `Restart openvpn-exporter` — рестарт service после bin/template change
* `Reload systemd` — при template change unit-файла
