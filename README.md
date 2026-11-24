# GPU LLM server with Ansible

Роль развёртывает Ollama с Qwen3.6-27B на Ubuntu Server 22.04/24.04 (amd64) и NVIDIA GPU: драйвер, Docker CE, NVIDIA Container Toolkit, контейнер, модель и проверки API/VRAM. Основной сценарий — сервер по SSH. WSL2 использован как тестовый стенд; установка драйвера и перезагрузка физического Ubuntu-сервера ещё требуют приёмки на целевом оборудовании. [Результаты проверки](VALIDATION.md).

## Подготовка сервера

Нужны доступная NVIDIA GPU, SSH, Python 3, systemd, пользователь с sudo и исходящий HTTPS к репозиториям Ubuntu, Docker, NVIDIA и Ollama. При Secure Boot может потребоваться ручное подтверждение MOK через консоль. Запланируйте окно обслуживания: установка драйвера может потребовать перезагрузку, настройка runtime — перезапуск Docker.

Qwen3.6-27B не помещается целиком в 8 ГБ VRAM. На тестовом RTX 3070 модель использовала CPU и RAM; для production подберите GPU, память и диск под требуемую нагрузку.

## Настройка и запуск

На управляющем Linux-узле:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install 'ansible-core==2.19.15'
ansible-galaxy collection install -r requirements.yml
cp config.example.yml config.local.yml
```

В `inventory.ini` замените пример адреса и пользователя. В `config.local.yml` укажите проверенный `gpu_llm_image` (`ollama/ollama@sha256:...` либо явная версия) и подходящий целевому серверу `gpu_llm_driver_package`. Для согласованной автоматической перезагрузки установите `gpu_llm_allow_reboot: true`; по умолчанию она запрещена. Локальный конфиг исключён из Git. Остальные параметры находятся в `roles/gpu_llm/defaults/main.yml`.

```bash
ansible -i inventory.ini gpu_servers -m ping
ansible-playbook -i inventory.ini playbook.yml -e @config.local.yml --ask-become-pass
ansible-playbook -i inventory.ini verify.yml -e @config.local.yml
```

Если sudo не требует пароля, уберите `--ask-become-pass`. Первый запуск загружает образ и веса и может занять много времени. Для приёмки повторите playbook (без изменения конфигурации ожидается `changed=0`), перезагрузите сервер в окно обслуживания и повторите `verify.yml`.

## API

Сервис слушает только `127.0.0.1:11434`. Для удалённого доступа откройте SSH-туннель на клиенте:

```bash
ssh -N -L 11434:127.0.0.1:11434 ubuntu@YOUR_SERVER
```

В другом терминале клиента:

```bash
curl --noproxy '*' --fail-with-body http://127.0.0.1:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.6:27b","prompt":"Объясни назначение Ansible в трёх предложениях.","stream":false,"think":false,"options":{"num_ctx":2048,"num_predict":128}}'
```

Ожидается JSON с непустым `response` и `done: true`. `verify.yml` дополнительно проверяет выделение VRAM; частичный CPU/GPU offload считается допустимым.

## Эксплуатация и путь в production

Docker запускается через systemd, контейнер использует `restart: unless-stopped`, веса хранятся в volume `gpu_llm_models`. После старта ОС API поднимается автоматически, а модель загружается при первом запросе. Состояние на сервере:

```bash
nvidia-smi
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml ps
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml logs --tail=100 ollama
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml exec -T ollama ollama ps
```

Перед обновлением сохраните прежние image/model digest и резервную копию volume. Для отката восстановите прежнюю конфигурацию, повторите playbook и `verify.yml`. `docker compose down -v` удаляет volume с весами. Healthcheck сам по себе не перезапускает зависший контейнер.

Перед production нужно добавить TLS-шлюз с аутентификацией и лимитами запросов, хранение секретов в Vault/secret manager, мониторинг GPU через DCGM Exporter/Prometheus/Grafana и внешние проверки доступности и времени ответа. Эти компоненты пока не устанавливаются. На физическом сервере отдельно проверьте установку драйвера, reboot, восстановление API и недоступность порта 11434 извне.

Локальный сценарий WSL2 приведён в [WSL-TEST.md](WSL-TEST.md).
