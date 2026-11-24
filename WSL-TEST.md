# Проверка роли в WSL2

Нужна Ubuntu 24.04 в WSL2 с systemd, рабочим nvidia-smi и NVIDIA-драйвером Windows. Linux NVIDIA driver внутри WSL не устанавливается. Используется Docker Engine внутри Ubuntu; отключите Docker Desktop integration для этого дистрибутива, если она включена.

Скопируйте новую версию в отдельный Linux-каталог, не поверх старой проверенной копии. Выполните установку зависимостей по README. Создайте config.local.yml из примера, укажите digest уже проверенного образа. Пакет Linux-драйвера оставьте пустым; gpu_llm_allow_reboot=false. Режим WSL задан в inventory.wsl.ini.

На стенде с 32 ГБ RAM хоста установлен лимит WSL 24 ГБ RAM и 8 ГБ swap. Не заменяйте существующий .wslconfig целиком: сохраните прочие настройки. Перед тестом проверьте free -h, df -h / /mnt/c и nvidia-smi.

```bash
source .venv/bin/activate
ansible-playbook -i inventory.wsl.ini playbook.yml -e @config.local.yml --syntax-check
ansible-playbook -i inventory.wsl.ini playbook.yml -e @config.local.yml --ask-become-pass
ansible-playbook -i inventory.wsl.ini playbook.yml -e @config.local.yml --ask-become-pass
ansible-playbook -i inventory.wsl.ini verify.yml -e @config.local.yml
```

Повторный deployment не должен заново скачивать веса или пересоздавать контейнер без изменения конфигурации. Сохраняется прежний project name gpu_llm и volume gpu_llm_models. Переход с latest на digest может один раз пересоздать контейнер.

В PowerShell выполните wsl --shutdown (останавливает все WSL-дистрибутивы), снова откройте Ubuntu и повторите verify.yml. Это проверяет старт сервиса при запуске WSL, но не старт WSL при загрузке Windows и не reboot физического сервера.

Ожидаются отсутствие failed/unreachable, непустой ответ и size_vram>0. Запрос API приведён в README. Логи не должны содержать секреты перед публикацией.

Старые переменные ollama_* и wsl_test_mode переименованы в gpu_llm_*; прежний локальный профиль напрямую не подходит. На первом запуске новой роли требуется config.local.yml с image digest. Результаты тестов исходной версии находятся в VALIDATION.md; новой роли необходим повторный GPU-прогон.
