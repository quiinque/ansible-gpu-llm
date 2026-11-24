# GPU-сервер с Ollama и Ansible

Проект устанавливает NVIDIA-драйвер, Docker CE и NVIDIA Container Toolkit, запускает Ollama, проверяет генерацию через API и выделение VRAM. Финальная модель — **Qwen3.6-27B**, выбирается профилем `-e @qwen36.yml` во всех командах ниже. Без профиля используется небольшая `qwen2.5:3b`. Целевая платформа: **Ubuntu Server 22.04/24.04, amd64, NVIDIA GPU**. Для локальной проверки предусмотрен отдельный режим WSL2; Jetson и Kubernetes не входят в область этого playbook.

Проверено на RTX 3070 8 ГБ, 32 ГБ RAM хоста и WSL с лимитом 24 ГБ RAM / 8 ГБ swap. Qwen3.6-27B использовала CPU и GPU, контекст 2048. В одном запросе скорость генерации составила около 4,1 токена/с. Это функциональный тест, не нагрузочный benchmark. См. [отчёт](VALIDATION.md).

## Архитектура

```text
Клиент → SSH-туннель → 127.0.0.1:11434 → Docker/Ollama → NVIDIA GPU
                                             ↓
                                   volume gpu_llm_models
```

Модель хранится в постоянном Docker volume. Docker включён в systemd, контейнер имеет `restart: unless-stopped` и healthcheck. После перезагрузки запускается API, а модель загружается в VRAM при первом запросе. Намеренно остановленный контейнер автоматически не запускается — повторите playbook или выполните `docker compose ... up -d`.

## Требования

- Выделенный сервер либо VM с PCI passthrough, SSH, Python 3, пользователь с sudo; включены Ubuntu repositories `restricted` и `universe`.
- NVIDIA GPU, поддерживаемый выбранной версией Ollama и драйвера. Для Qwen3.6-27B на GPU с 8 ГБ VRAM нужна оперативная память для частичного CPU offload: фактически проверен стенд, указанный выше. Для образов и весов оставьте запас диска; размер загрузки не равен расходу памяти. Требования зависят от модели, контекста и параллелизма.
- Исходящий HTTPS к Ubuntu mirrors, download.docker.com, nvidia.github.io, Docker Hub, registry.ollama.ai и адресам хранения моделей, на которые они перенаправляют.
- Управляющая машина: Linux/macOS либо Linux в WSL, Python 3.11+, Ansible Core 2.18–2.19. Нативный Windows не используется как Ansible controller.
- Перед запуском проверьте совместимость GPU. При Secure Boot модуль должен быть подписан доверенным ключом: enrollment MOK через консоль может потребовать ручной подготовки. Playbook не отключает Secure Boot.

Playbook предназначен для выделенного узла: установка драйвера может потребовать перезагрузки, настройка runtime — перезапуска Docker с влиянием на другие контейнеры. По умолчанию `allow_reboot: true`; перезагрузка выполняется также при существующем `/var/run/reboot-required`. Конфликтующие Docker-пакеты не удаляются автоматически: выполнение остановится с сообщением.

## Запуск

Распакуйте проект на управляющей машине и перейдите в каталог проекта.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install 'ansible-core>=2.18,<2.20'
ansible-galaxy collection install -r requirements.yml
```

Замените адрес-заглушку и пользователя в `inventory.ini`. Настройте SSH-ключ, подключитесь вручную и проверьте fingerprint сервера; проверка host key включена. Пароли и приватные ключи в проект не помещайте.

```bash
ssh ubuntu@YOUR_SERVER
# Выйдите из SSH, далее команды на controller:
ansible gpu_servers -m ping
ansible-playbook playbook.yml -e @qwen36.yml --syntax-check
ansible-playbook playbook.yml -e @qwen36.yml --ask-become-pass
```

Для passwordless sudo уберите `--ask-become-pass`. Первый запуск может занять десятки минут из-за драйвера, перезагрузки и загрузки модели. В профиле Qwen3.6 загрузка модели ограничена четырьмя часами, время изменяется через `model_pull_timeout`. Фактический срок зависит от сети и оборудования.

Основные настройки — `group_vars/gpu_servers.yml`:

| Переменная | Назначение |
| --- | --- |
| `ollama_image` | Образ; для демонстрации `latest`, для production задайте проверенный tag или digest |
| `ollama_model` | Имя модели с явным тегом; по умолчанию `qwen2.5:3b` |
| `ollama_context_length` | Контекст, по умолчанию 4096 |
| `nvidia_driver_package` | Пусто: сохранить рабочий драйвер либо выбрать compute-драйвер Ubuntu; можно указать пакет явно |
| `allow_reboot` | Разрешать необходимую перезагрузку; при false playbook остановится, если она нужна |
| `refresh_model` | Принудительно обновить существующий тег модели |
| `require_gpu_inference` | Требовать VRAM > 0 после генерации, по умолчанию true |

Рабочий существующий драйвер не обновляется автоматически. CUDA Toolkit на хосте не устанавливается: для контейнерного inference используются драйвер и NVIDIA Container Toolkit. Docker daemon.json изменяется через `nvidia-ctk`, прочие настройки сохраняются.

Повторный запуск сохраняет модель и конфигурацию. Образы загружаются только при отсутствии локально; существующая модель скачивается повторно только при `refresh_model=true`. Проверки inference выполняются при каждом запуске и временно загружают модель в память. Режим `--check` не заменяет развёртывание: runtime, контейнер и API ещё могут отсутствовать.

## API-запрос

Для локального теста в WSL2 сначала выполните инструкции в [WSL-TEST.md](WSL-TEST.md). Там SSH-туннель не нужен: запрос отправляется из той же Ubuntu в WSL.

На клиентской машине откройте SSH-туннель и оставьте его работающим:

```bash
ssh -N -L 11434:127.0.0.1:11434 ubuntu@YOUR_SERVER
```

В другом терминале клиента (bash):

```bash
curl --noproxy '*' --fail-with-body http://127.0.0.1:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.6:27b","prompt":"Объясни, что такое нейросеть, в двух предложениях.","stream":false,"think":false,"options":{"num_ctx":2048,"num_predict":128}}'
```

Ожидается HTTP 200 и JSON с непустым `response`, `done: true` и положительным `eval_count`. Содержимое ответа недетерминированно. На самом сервере используйте тот же URL без туннеля.

## Проверка и приёмка на GPU

Playbook автоматически проверяет `nvidia-smi` на хосте и внутри контейнера, доступность `/api/tags`, генерацию и `/api/ps`. Проверка `size_vram > 0` подтверждает размещение хотя бы части модели на GPU; она не гарантирует 100% GPU offload.

Повторная проверка API без установки пакетов:

```bash
ansible-playbook verify.yml -e @qwen36.yml
```

На сервере:

```bash
nvidia-smi
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml ps
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml exec -T ollama ollama ps
curl --noproxy '*' --fail http://127.0.0.1:11434/api/ps
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml logs --tail=100 ollama
```

Критерии приёмки:

1. Основной playbook заканчивается без failed/unreachable, контейнер healthy, генерация непустая, у нужной модели `size_vram > 0`.
2. Повторный запуск не переустанавливает рабочий драйвер, не перезагружает узел без причины и не скачивает модель повторно. Обновление APT cache может отображаться как изменение.
3. В согласованное окно перезагрузите сервер (`sudo reboot`), дождитесь SSH, выполните `verify.yml`: API должен восстановиться без ручного запуска контейнера.
4. С другой машины без SSH-туннеля запрос к `http://SERVER_IP:11434/api/tags` не должен проходить; через туннель — должен.

**Статус проверки:** Qwen3.6-27B проверена пользователем на Ubuntu 24.04 в WSL2 с RTX 3070: API, повторный запуск с changed=0 и восстановление после перезапуска WSL. Серверная установка драйвера и reboot физического сервера не проверялись. Подробности — [VALIDATION.md](VALIDATION.md).

## Диагностика

| Симптом | Что проверить |
| --- | --- |
| `nvidia-smi` не работает | Secure Boot/MOK, `lsmod`, журнал ядра `journalctl -k`, совместимость драйвера; после исправления повторить запуск |
| Не найдены headers или driver package | Поддерживаемое ядро и Ubuntu repositories restricted/universe |
| Контейнер не видит GPU | `docker info`, NVIDIA runtime в daemon.json, версии driver/toolkit, журналы Docker |
| API работает, VRAM = 0 | Совместимость GPU, логи Ollama, размер модели; CPU-only не считается успешной GPU-проверкой |
| OOM или timeout | Уменьшить модель/контекст, освободить GPU, увеличить generation_timeout при медленном старте |
| Загрузка модели не завершена | DNS, HTTPS, свободное место `df -h`, доступность registry; исправить и повторить playbook |

Healthcheck проверяет процесс/API через `ollama list`, но не запускает генерацию. Docker не перезапускает контейнер только из-за статуса unhealthy; restart policy действует при выходе процесса. Для зависаний нужен внешний мониторинг и отдельная политика восстановления.

## Обновления и данные

Для управляемого обновления замените `ollama_image` на новый проверенный digest и повторите playbook. Для обновления весов выполните `ansible-playbook playbook.yml -e @qwen36.yml -e refresh_model=true`. Mutable tags не обеспечивают воспроизводимость: фиксируйте image digest, версию коллекции, пакетов и digest модели из `/api/tags` в журнале релиза; перед обновлением сохраняйте volume. Веса и права использования модели следует проверять для конкретного применения.

`docker compose -p gpu_llm -f /opt/ollama/compose.yml down` останавливает сервис, сохраняя данные; `down -v` удаляет веса. Для резервной копии остановите сервис в согласованное окно и сохраните volume `gpu_llm_models` вместе с Compose и переменными. После восстановления выполните `up -d` и `verify.yml`.

## Улучшения для production

- Разместить TLS reverse proxy/API gateway перед localhost API: проверка токенов или mTLS, ограничения числа запросов, размера тела и времени выполнения. Разрешить клиентам только inference endpoints; операции загрузки/удаления моделей оставить администраторам. Секреты хранить в Ansible Vault или secret manager. Сейчас защита удалённого доступа обеспечивается SSH, встроенная API-аутентификация не настроена.
- Добавить NVIDIA DCGM Exporter → Prometheus → Grafana: VRAM, GPU utilization, temperature, power, Xid errors. Endpoint метрик оставить во внутренней сети; совместимость DCGM с конкретным GPU проверить заранее. Добавить внешний HTTP probe, редкую проверку генерации и алерты на latency/5xx/OOM/disk usage.
- Для нескольких узлов — gateway и отдельный model volume/cache на каждом узле, последовательные обновления с выводом узла из балансировки. Для повышенной конкурентности отдельно сравнить Ollama и vLLM на своей модели и нагрузке.

## Официальная документация

- [Ubuntu: установка NVIDIA-драйверов](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/)
- [Docker Engine на Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- [Ollama в Docker](https://docs.ollama.com/docker)
- [Ollama Generate API](https://docs.ollama.com/api/generate)
- [Ollama: список загруженных моделей и VRAM](https://docs.ollama.com/api/ps)
- [NVIDIA DCGM Exporter](https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/dcgm-exporter.html)

