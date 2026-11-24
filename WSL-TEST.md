# Локальная проверка на Windows 11 / WSL2

Этот режим проверяет Docker, NVIDIA Container Toolkit, модель и API на собственной GPU. Установка драйвера Ubuntu и перезагрузка физического сервера остаются отдельными непроверенными этапами. GPU предоставляется драйвером Windows; Linux-драйвер NVIDIA внутри WSL устанавливать нельзя ([NVIDIA](https://docs.nvidia.com/cuda/wsl-user-guide/index.html)).

Для Qwen3.6-27B во всех командах ниже используется профиль `-e @qwen36.yml`. Проверенный стенд: RTX 3070 8 ГБ, 32 ГБ RAM хоста, WSL с лимитом 24 ГБ RAM и 8 ГБ swap. Модель использует CPU и GPU. Перед запуском проверьте `free -h` и `df -h / /mnt/c`. Настройка лимита WSL выполняется через существующий файл Windows `%UserProfile%\.wslconfig` (раздел `[wsl2]`, параметры `memory=24GB` и `swap=8GB`), сохраняя прочие настройки; затем требуется остановка и повторный запуск WSL.

## Подготовка

Нужна Ubuntu 24.04 в WSL2, работающий `nvidia-smi` и systemd. В Ubuntu:

```bash
nvidia-smi
ps -p 1 -o comm=
```

Вторая команда должна показать `systemd`. Иначе включите его по [инструкции Microsoft](https://learn.microsoft.com/en-us/windows/wsl/systemd): добавьте `systemd=true` в раздел `[boot]` существующего `/etc/wsl.conf`, сохранив остальные настройки, затем выполните `wsl --shutdown` в PowerShell и снова откройте Ubuntu. Команда остановит все WSL-дистрибутивы: сначала завершите их текущую работу.

Используется Docker Engine внутри Ubuntu, не Docker Desktop. Если Docker Desktop уже установлен, отключите его интеграцию с этой Ubuntu и завершите Docker Desktop перед тестом. Не запускайте два Docker daemon через один socket. Не устанавливайте отдельно `docker.io`: playbook устанавливает Docker CE из официального репозитория.

Скопируйте обновлённый каталог проекта в Linux filesystem, например `~/gpu-llm-wsl`, и перейдите в него. Запуск из `/mnt/c` может мешать правам доступа и чтению ansible.cfg. Файл `inventory.ini` редактировать не надо: используется отдельный inventory для localhost.

## Установка Ansible и проверка синтаксиса

Все следующие команды выполняются в Ubuntu WSL, в каталоге проекта:

```bash
sudo apt-get update && sudo apt-get install -y python3-venv python3-pip
python3 -m venv .venv
source .venv/bin/activate
pip install 'ansible-core>=2.18,<2.20'
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory.wsl.ini playbook.yml -e @qwen36.yml --syntax-check
ansible-playbook -i inventory.wsl.ini verify.yml -e @qwen36.yml --syntax-check
```

Продолжайте только если команды завершились успешно.

## Развёртывание

```bash
set -o pipefail
ansible-playbook -i inventory.wsl.ini playbook.yml -e @qwen36.yml --ask-become-pass | tee wsl-deploy.log
```

На запрос `BECOME password` введите пароль пользователя Ubuntu. Устанавливаются пакеты и системные сервисы внутри WSL; скачиваются образ и веса модели. Драйвер Windows не изменяется. В режиме WSL пропускаются PCI detection, установка Linux headers/драйвера и reboot. Вместо PCI проверяется `/dev/dxg`; `nvidia-smi` вызывается по абсолютному пути WSL, доступному и при sudo.

Режим WSL включён только через `inventory.wsl.ini`. Playbook отклоняет случайный запуск серверного режима в WSL и WSL-режима на обычном Linux. Системный Docker socket выбран явно. Если основной playbook завершается с ошибкой, сохраните сообщение и исправьте причину; не отключайте GPU-проверку ради успешного отчёта.

## Проверка API и повторного запуска

```bash
curl --noproxy '*' --fail-with-body http://127.0.0.1:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.6:27b","prompt":"Объясни, что такое нейросеть, в двух предложениях.","stream":false,"think":false,"options":{"num_ctx":2048,"num_predict":128}}'

curl --noproxy '*' --fail http://127.0.0.1:11434/api/ps
sudo docker compose -p gpu_llm -f /opt/ollama/compose.yml exec -T ollama ollama ps
ansible-playbook -i inventory.wsl.ini verify.yml -e @qwen36.yml | tee wsl-verify.log
ansible-playbook -i inventory.wsl.ini playbook.yml -e @qwen36.yml --ask-become-pass | tee wsl-repeat.log
```

Требуется непустой ответ, `done: true`, `eval_count > 0` и `size_vram > 0` у нужной модели. Playbook проверяет эти условия автоматически. RTX 3070 имеет 8 ГБ VRAM; часть памяти занимает Windows, поэтому перед тестом закройте игры и другие GPU-нагрузки. Если не хватает памяти, уменьшите контекст или модель.

Для проверки восстановления: завершите работу в WSL, выполните `wsl --shutdown` в PowerShell, откройте Ubuntu, перейдите в проект, активируйте `.venv` и повторите `verify.yml`. Это проверка старта сервисов при запуске WSL. Она не подтверждает автоматический запуск WSL при загрузке Windows и не заменяет reboot-тест Ubuntu Server.

## Что приложить к сдаче

- Архив проекта, README и этот сценарий.
- Логи deployment, повторного запуска, verify; вывод `nvidia-smi`, `ollama ps` и API без личных данных.
- Укажите фактический стенд: Windows 11 → WSL2 → Ubuntu → Docker → RTX 3070.
- Отдельно укажите, что установка NVIDIA-драйвера и reboot на физическом Ubuntu Server не тестировались.

Перед отправкой проверьте логи на наличие имён пользователей, путей, адресов и иной информации, которую не хотите публиковать.


