# NVIDIA GPU inference with Ansible

Автоматизация Ubuntu Server 22.04/24.04 amd64: NVIDIA driver → Docker CE → NVIDIA Container Toolkit → Ollama → Qwen3.6-27B API.

Роль `gpu_llm` использует defaults, handlers, шаблон Compose, проверку аргументов и отдельные этапы. Основной сценарий — сервер по SSH; WSL2 — дополнительный тестовый стенд. См. [WSL-TEST.md](WSL-TEST.md) и [VALIDATION.md](VALIDATION.md).

CI и новая роль прошли проверку: deployment, повторный запуск с changed=0, API/VRAM после перезапуска WSL. Серверная установка драйвера и native reboot остаются непроверенными. Подробности и ссылка на CI — в [VALIDATION.md](VALIDATION.md).

## Быстрый старт на Ubuntu Server

На Linux controller с Python 3.12:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
cp config.example.yml config.local.yml
```

Заполните `config.local.yml`:

- `gpu_llm_image`: проверенный `ollama/ollama@sha256:...` (предпочтительно) либо явный числовой version tag. `latest` запрещён проверкой конфигурации.
- `gpu_llm_driver_package`: пакет драйвера, выбранный под GPU/ОС. Используйте рекомендации Ubuntu/NVIDIA; намеренно нет универсального номера для любого GPU.
- `gpu_llm_allow_reboot`: по умолчанию false. При необходимости reboot playbook остановится после установки пакетов с объяснением. Можно перезагрузить сервер вручную и повторить запуск или разрешить reboot в согласованное окно.

`config.local.yml` исключён из Git. Пример не содержит рабочего image digest: его выбирает оператор, исходя из проверенной версии. Версию уже протестированного образа можно узнать на рабочем хосте:

```bash
sudo docker image inspect ollama/ollama:latest --format '{{index .RepoDigests 0}}'
```

Если образ устанавливался под другим тегом, подставьте его. Это не скачивает и не обновляет образ. Убедитесь, что digest относится к образу, используемому работающим контейнером.

Замените адрес-заглушку в `inventory.ini`, настройте SSH-ключ и sudo. Host key checking включён: сначала подключитесь вручную и проверьте fingerprint. Предполагаются Python 3, systemd и Ubuntu repositories restricted/universe на target.

```bash
ansible -i inventory.ini gpu_servers -m ping
ansible-playbook -i inventory.ini playbook.yml -e @config.local.yml --syntax-check
ansible-playbook -i inventory.ini playbook.yml -e @config.local.yml --ask-become-pass
ansible-playbook -i inventory.ini verify.yml -e @config.local.yml
```

Для passwordless sudo уберите `--ask-become-pass`. Qwen3.6-27B теперь задана в defaults роли; `qwen36.yml` оставлен как необязательный явный профиль. Для другой модели переопределите `gpu_llm_model` в локальной конфигурации.

## Структура

```text
playbook.yml                 # serial deployment, остановка при ошибке узла
verify.yml                   # отдельная проверка API и VRAM
roles/gpu_llm/
  defaults/main.yml          # переопределяемые параметры gpu_llm_*
  meta/argument_specs.yml    # типы публичных параметров
  tasks/                     # preflight, driver, docker, deploy, verify
  handlers/main.yml          # перезапуск Docker при изменениях
  templates/compose.yml.j2   # декларативный сервис
inventory.ini                # пример серверного inventory
inventory.wsl.ini            # локальный WSL без Linux driver/reboot
config.example.yml           # обязательные настройки оператора
.github/workflows/validate.yml
```

## Поведение и ограничения

- Все Ansible-модули названы через FQCN. Модули apt/file/template/systemd/Compose управляют состоянием; command используется для NVIDIA CLI, диагностики и pull модели. Read-only команды имеют changed_when=false; pull выполняется при отсутствии модели или явном refresh.
- Установка драйвера выбирается явно. Нет автоматического удаления конфликтующих Docker-пакетов, смены драйвера на рекомендованный без настройки или отключения Secure Boot. MOK enrollment может требовать ручной подготовки.
- Docker runtime настраивается через nvidia-ctk с сохранением прочих настроек daemon.json. Restart Docker выполняет handler, до создания GPU-контейнера. Обновление пакетов может затронуть другие контейнеры: выделенный узел и окно обслуживания обязательны для такого изменения.
- `state: present` сохраняет установленные версии пакетов. Для точного package pinning задайте `gpu_llm_container_packages_to_install` с `name=version` и при необходимости версию driver package. Версии зависят от Ubuntu release и источников; универсальный lock для всех серверов не заявляется.
- Ansible Core, линтеры и community.docker зафиксированы в requirements. Image digest обеспечивает неизменяемый образ; model tag может меняться. Существующая модель не обновляется без `gpu_llm_refresh_model: true`; фиксируйте её digest из `/api/tags` в журнале релиза.
- `--check --diff` предназначен для частичного просмотра изменений пакетов/файлов, а не проверки чистой установки. Runtime, deployment и inference пропускаются; новые репозитории/пакеты могут быть недоступны в dry-run. Полная приёмка — реальный прогон на тестовом узле.
- Теги `driver`, `docker`, `deploy`, `verify` нужны для обслуживания уже подготовленного узла. Первый запуск выполняйте целиком; `verify.yml` предпочтителен для отдельного теста без установки пакетов.

## API и автозапуск

Порт привязан только к `127.0.0.1:11434`. Удалённый клиент открывает SSH-туннель:

```bash
ssh -N -L 11434:127.0.0.1:11434 ubuntu@YOUR_SERVER
```

В другом терминале (на самом сервере/WSL туннель не нужен):

```bash
curl --noproxy '*' --fail-with-body http://127.0.0.1:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.6:27b","prompt":"Объясни назначение Ansible в трёх предложениях.","stream":false,"think":false,"options":{"num_ctx":2048,"num_predict":128}}'
```

Ожидается HTTP 200, непустой response, done=true. verify.yml проверяет также eval_count>0 и выделение VRAM у запрошенной модели; это допускает CPU/GPU offload.

Docker включён в systemd; контейнер использует restart: unless-stopped. Веса сохраняются в volume `gpu_llm_models`. После старта ОС запускается API, веса загружаются при первом запросе. Намеренно остановленный контейнер автоматически не восстанавливается. Healthcheck выполняет ollama list, но unhealthy сам по себе не вызывает restart: внешнее восстановление зависаний не реализовано.

## Ресурсы и проверка

Для Qwen3.6-27B с 8 ГБ VRAM используется частичный offload в RAM/CPU. Новая роль проверена на RTX 3070, 32 ГБ RAM хоста и WSL с лимитом 24 ГБ RAM / 8 ГБ swap, контекст 2048. В одном запросе получено около 4,1 токена/с; это не SLA. Нужен свободный диск для Docker и весов, исходящий HTTPS к Ubuntu mirrors, Docker, NVIDIA и Ollama registry/storage.

```bash
yamllint .
ansible-lint
ansible-playbook -i inventory.ini playbook.yml --syntax-check
ansible-playbook -i inventory.wsl.ini playbook.yml --syntax-check
```

CI выполняет статические проверки, но не имеет GPU. После реального deployment повторите playbook (ожидается changed=0, кроме возможного обновления apt cache), проверьте verify.yml, reboot и повторный verify. С другой машины проверьте недоступность SERVER_IP:11434 без туннеля.

Диагностика на target:

```bash
nvidia-smi
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml ps
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml logs --tail=100 ollama
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml exec -T ollama ollama ps
```

При OOM уменьшите модель/контекст, проверьте free -h, df -h и GPU usage. Во время первого pull Ansible может долго не показывать прогресс образа. Ошибки driver/Secure Boot/runtime нужно устранить, а не отключать проверку GPU.

## Обновления, защита и мониторинг

Меняйте image digest после проверки новой версии на стенде. Для отката восстановите прежнюю конфигурацию, digest и при необходимости backup volume, затем повторите deployment и verify. `docker compose down` сохраняет volume; `down -v` удаляет веса. Backup volume делайте при остановленном сервисе в окно обслуживания.

Для production предлагаются TLS gateway с токенами или mTLS, rate limits и ограничением доступных endpoints; секреты через Ansible Vault/secret manager. В текущем проекте удалённый доступ защищён SSH, отдельной API-аутентификации нет.

GPU-мониторинг: DCGM Exporter → Prometheus → Grafana с проверкой совместимости GPU, приватным endpoint и алертами на VRAM/temperature/Xid errors. Дополнительно HTTP probe, периодическая генерация, latency, disk и OOM alerts. Мониторинг предложен как расширение, не установлен автоматически.

## Документация

- [Ansible roles и defaults](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Ansible Lint](https://docs.ansible.com/projects/lint/usage/)
- [Ubuntu NVIDIA drivers](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/)
- [Docker Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- [Ollama Docker](https://docs.ollama.com/docker)
