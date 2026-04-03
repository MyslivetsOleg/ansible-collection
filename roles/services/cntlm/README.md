# Ansible Role: cntlm

Роль устанавливает и настраивает [CNTLM](http://cntlm.sourceforge.net/) — локальный HTTP-прокси с поддержкой NTLM/NTLMv2-аутентификации. Установка из локального RPM-файла, управление сервисом через systemd.

---

## Требования

- RHEL / CentOS / Rocky Linux / AlmaLinux 7, 8 или 9
- Ansible ≥ 2.12
- RPM-файл `cntlm` в директории `files/` роли

---

## Структура роли

```
cntlm/
├── defaults/
│   └── main.yml              # Все переменные с дефолтами
├── files/
│   └── cntlm.rpm             # Положите сюда RPM-пакет
├── handlers/
│   └── main.yml              # reload systemd, restart cntlm
├── meta/
│   └── main.yml              # Метаданные роли
├── tasks/
│   ├── main.yml              # Точка входа
│   ├── validate.yml          # Проверка обязательных переменных
│   ├── install.yml           # Копирование и установка RPM
│   ├── configure.yml         # Деплой cntlm.conf из шаблона
│   ├── service.yml           # Деплой systemd unit и старт сервиса
│   └── verify.yml            # Проверка работоспособности прокси
├── templates/
│   ├── cntlm.conf.j2         # Шаблон конфигурационного файла
│   └── cntlm.service.j2      # Шаблон systemd unit-файла
└── README.md
```

---

## Быстрый старт

### 1. Положите RPM в директорию роли

```bash
cp /path/to/cntlm-*.rpm roles/cntlm/files/cntlm.rpm
```

### 2. Сгенерируйте NTLMv2-хэш пароля

На любой машине с установленным cntlm:

```bash
cntlm -H -u ваш_логин -d ВАШ_ДОМЕН
```

Из вывода нужна только строка `PassNTLMv2`.

### 3. Сохраните хэш в Ansible Vault

```bash
ansible-vault encrypt_string 'ВАШХЭШЗДЕСЬ' --name 'cntlm_ntlm_hash'
```

Скопируйте вывод в `group_vars` или `host_vars`.

### 4. Настройте переменные

```yaml
# group_vars/all.yml
cntlm_username: "ivanov"
cntlm_domain: "CORP"
cntlm_ntlm_hash: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...

cntlm_proxy_list:
  - "proxy.corp.example.com:8080"

cntlm_listen_port: 3128
```

### 5. Подключите роль в плейбуке

```yaml
# playbook.yml
- name: Setup CNTLM proxy
  hosts: workstations
  roles:
    - role: cntlm
```

### 6. Запустите

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

---

## Переменные

### Установка

| Переменная | По умолчанию | Описание |
|---|---|---|
| `cntlm_rpm_src` | `files/cntlm.rpm` | Путь к RPM относительно директории роли |
| `cntlm_bin_path` | `/usr/sbin/cntlm` | Путь к бинарнику после установки |

### Аутентификация

| Переменная | По умолчанию | Описание |
|---|---|---|
| `cntlm_username` | `""` | Логин (обязательно) |
| `cntlm_domain` | `""` | Домен (обязательно) |
| `cntlm_ntlm_hash` | `""` | NTLMv2-хэш (обязательно, используйте Vault) |

### Upstream-прокси

| Переменная | По умолчанию | Описание |
|---|---|---|
| `cntlm_proxy_list` | `["proxy.example.com:8080"]` | Список upstream NTLM-прокси |

### Локальный сервер

| Переменная | По умолчанию | Описание |
|---|---|---|
| `cntlm_listen_address` | `127.0.0.1` | Адрес |
| `cntlm_listen_port` | `3128` | Порт |
| `cntlm_conf_path` | `/etc/cntlm.conf` | Путь к конфигу |

### Systemd

| Переменная | По умолчанию | Описание |
|---|---|---|
| `cntlm_service_name` | `cntlm` | Имя unit |
| `cntlm_service_enabled` | `true` | Автозапуск |
| `cntlm_service_state` | `started` | Желаемое состояние |
| `cntlm_service_user` | `cntlm` | Системный пользователь |
| `cntlm_service_group` | `cntlm` | Системная группа |

### Проверка

| Переменная | По умолчанию | Описание |
|---|---|---|
| `cntlm_run_check` | `true` | Проверять прокси после старта |
| `cntlm_check_url` | `http://example.com` | URL для тестового запроса |
| `cntlm_wait_timeout` | `15` | Таймаут ожидания порта (сек) |

### Дополнительные параметры конфига

| Переменная | По умолчанию | Описание |
|---|---|---|
| `cntlm_timeout` | `3000` | Таймаут upstream (мс) |
| `cntlm_max_keepalive` | `500` | Макс. keep-alive соединений |
| `cntlm_dns_ttl` | `0` | TTL DNS-кэша (0 = отключён) |

---

## Теги

```bash
# Только валидация переменных
ansible-playbook playbook.yml --tags cntlm_validate

# Только переустановить RPM
ansible-playbook playbook.yml --tags cntlm_install

# Только обновить конфиг (сервис перезапустится если конфиг изменился)
ansible-playbook playbook.yml --tags cntlm_configure

# Только пересоздать systemd unit
ansible-playbook playbook.yml --tags cntlm_service

# Только проверить что прокси работает
ansible-playbook playbook.yml --tags cntlm_verify
```

---

## Идемпотентность

- RPM копируется и устанавливается только если бинарник `/usr/sbin/cntlm` отсутствует
- Конфиг и unit-файл деплоятся только при изменении шаблона или переменных
- Сервис перезапускается только через handler — то есть только при реальных изменениях
- Повторный запуск роли без изменений переменных не производит никаких действий

---

## Безопасность

- `cntlm.conf` создаётся с правами `0640` (root:cntlm) — хэш пароля недоступен другим пользователям
- Сервис запускается от системного пользователя `cntlm` без login shell
- Systemd unit включает: `NoNewPrivileges`, `PrivateTmp`, `ProtectSystem=strict`, `ProtectHome`
- NTLMv2-хэш рекомендуется хранить в Ansible Vault

---

## Лицензия

MIT
