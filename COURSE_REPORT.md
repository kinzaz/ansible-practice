# Ansible — подробный конспект курса и описание выполненной работы

Дата фиксации — **9 ноября 2026 года**, Europe/Moscow.

Документ предназначен для повторения курса после перерыва. В нём описаны понятия, последовательность упражнений, устройство проекта, зависимости между файлами, команды запуска, устранённые ошибки и границы полученного результата.

При подготовке отчёта репозиторий `https://github.com/kinzaz/ansible-practice.git` получен в локальный каталог `/Users/kinzaz/Documents/ansible-practice`. Проверенный коммит — `68e7bc93f4805988ff6478117189c7ba503440f4`, `Add initial administrator bootstrap playbook`. Содержимое отчёта сверено с обычными файлами этой копии и с результатами обучения в чате. Удалённые VM повторно не проверялись. У Vault-файла проверено наличие зашифрованного заголовка, расшифровка не выполнялась.

Секреты и пароли в документ не включены. Имена переменных с секретами приведены для объяснения устройства проекта. Команды установки, перезагрузки и применения playbook меняют состояние серверов. Их следует выполнять в контексте соответствующего упражнения.

## Содержание

1. [Цель курса и полученный результат](#1-цель-курса-и-полученный-результат)
2. [Связь с предыдущим курсом Docker и Jenkins](#2-связь-с-предыдущим-курсом-docker-и-jenkins)
3. [Серверы и места выполнения команд](#3-серверы-и-места-выполнения-команд)
4. [Установка Ansible и Python-окружение](#4-установка-ansible-и-python-окружение)
5. [SSH и цепочка доверия](#5-ssh-и-цепочка-доверия)
6. [Inventory и выбор серверов](#6-inventory-и-выбор-серверов)
7. [Первый удалённый модуль](#7-первый-удалённый-модуль)
8. [Playbook и формат YAML](#8-playbook-и-формат-yaml)
9. [Желаемое состояние и идемпотентность](#9-желаемое-состояние-и-идемпотентность)
10. [Изменения файлов и check mode](#10-изменения-файлов-и-check-mode)
11. [Переменные и подстановка значений](#11-переменные-и-подстановка-значений)
12. [Group vars и связь имён](#12-group-vars-и-связь-имён)
13. [Facts и условия](#13-facts-и-условия)
14. [Пакеты и повышение прав](#14-пакеты-и-повышение-прав)
15. [Handlers и порядок выполнения](#15-handlers-и-порядок-выполнения)
16. [Шаблоны Jinja2](#16-шаблоны-jinja2)
17. [Роли и окончательная структура проекта](#17-роли-и-окончательная-структура-проекта)
18. [Роль server_base](#18-роль-server_base)
19. [Роль docker](#19-роль-docker)
20. [Роль jenkins](#20-роль-jenkins)
21. [Роль jenkins_agent](#21-роль-jenkins_agent)
22. [Ansible Vault и разные виды секретов](#22-ansible-vault-и-разные-виды-секретов)
23. [Проверка агента через Jenkins API](#23-проверка-агента-через-jenkins-api)
24. [Register, until, assert и loop](#24-register-until-assert-и-loop)
25. [Tags и ansible.cfg](#25-tags-и-ansiblecfg)
26. [Управление SSH-ключами](#26-управление-ssh-ключами)
27. [Bootstrap и обычная настройка](#27-bootstrap-и-обычная-настройка)
28. [Git, deploy key и восстановление проекта](#28-git-deploy-key-и-восстановление-проекта)
29. [Повторный запуск и перезагрузка](#29-повторный-запуск-и-перезагрузка)
30. [Разобранные ошибки и их причины](#30-разобранные-ошибки-и-их-причины)
31. [Границы автоматизации и текущие нюансы](#31-границы-автоматизации-и-текущие-нюансы)
32. [Практический справочник команд](#32-практический-справочник-команд)
33. [Порядок подготовки новой VM](#33-порядок-подготовки-новой-vm)
34. [Словарь и вопросы для самопроверки](#34-словарь-и-вопросы-для-самопроверки)

## 1. Цель курса и полученный результат

После курса Docker/Jenkins пользователь захотел освоить Ansible на отдельной управляющей VM. Старый сервер Jenkins был удалён и заменён пустой VM. Сервер приложения Todo сохранился.

В предыдущем курсе автоматизировали выпуск приложения. В этом курсе автоматизировали подготовку сервера, на котором будут выполняться сборки. Ранее вручную устанавливали Docker, Java, настраивали swap, запускали контроллер и службу агента. Теперь эти действия описаны в YAML и ролях Ansible.

Получился проект с отдельными ролями подготовки системы, Docker, контроллера Jenkins и агента. Подключение выполняется по SSH под обычным пользователем, административные действия используют sudo. Секреты хранятся в зашифрованном Vault-файле. Проект сохранён на GitHub и проверен после клонирования в другой каталог.

Подтверждённые результаты включают запуск модулей на удалённом сервере, повторяемость настройки, восстановление после перезагрузки и проверку готовности агента через API Jenkins. Повторный запуск сохранял существующий Java-процесс, когда управляемые файлы не менялись.

Полную установку последней версии проекта на ещё одну полностью чистую VM отдельно не репетировали. Восстановление из Git проверено на существующей управляющей VM и уже настроенном сервере.

## 2. Связь с предыдущим курсом Docker и Jenkins

| Уровень работы | Предыдущий курс | Этот курс |
|---|---|---|
| Подготовка Ubuntu | Ручные команды | Задачи Ansible |
| Создание администратора | adduser, usermod, authorized_keys | user и authorized_key, bootstrap |
| Docker | Ручная установка репозитория и пакетов | Роль docker |
| Java | apt install | apt внутри server_base |
| Swap | fallocate, mkswap, swapon, fstab | Проверки и управляемые задачи |
| Контроллер Jenkins | Ручной Compose-файл и up | Шаблон и docker_compose_v2 |
| Агент | Команда Java и ручной unit | Роль jenkins_agent и systemd-шаблон |
| Секрет агента | Файл, созданный вручную | Vault и защищённая copy-задача |
| Проверка готовности | Просмотр UI и журналов | HTTP API с ожиданием online |
| Хранение конфигурации | Разные файлы на VM | Git-репозиторий Ansible |

Docker продолжает запускать контейнер Jenkins. Systemd продолжает управлять процессом агента. Ansible описывает желаемое состояние этих компонентов и применяет его при вызове playbook.

Уже существующий Pipeline выпуска Todo на старом Jenkins не переносился автоматически на новый контроллер в рамках этого курса. Исходники приложения и прежние отчёты сохранились отдельно.

## 3. Серверы и места выполнения команд

### Управляющий узел Ansible

- IP — `179.236.41.124`.
- Hostname — `nsk-1-vm-cwbc`.
- Ubuntu 26.04.1 LTS, архитектура x86_64.
- При первом осмотре — 1 vCPU, около 889 МиБ RAM, диск 14 ГБ, свободно около 12 ГБ.
- Python — 3.14.4.
- Пользователь — `ansible`, член группы sudo.
- Рабочий проект — `/home/ansible/ansible-practice`.
- Проверочная копия из Git — `/home/ansible/ansible-practice-restore-check`.

### Управляемая VM Jenkins

- IP — `179.236.42.247`.
- Hostname — `nsk-1-vm-7wq8`.
- Ubuntu 26.04, x86_64.
- Facts показали около 1895 МиБ RAM.
- Python — 3.14.4.
- Первоначально использовали root, затем пользователя devops с sudo.
- Контроллер Jenkins расположен в Compose-проекте `/opt/jenkins`.
- Агент работает на хосте под пользователем `jenkins-agent`.

### Существующее приложение

VM Todo `177.203.59.227` сохранена. В текущем inventory она не добавлена и playbook этого курса её не настраивают. Примеры группы application в объяснениях служили иллюстрацией формата.

### Mac

Mac использовался для SSH-подключений к управляющей VM, браузера Jenkins и туннеля. Основная установка и выполнение Ansible происходили на отдельном сервере.

```text
Mac
  → SSH → управляющая VM Ansible
                → SSH → VM Jenkins
                           → Docker → контроллер Jenkins
                           → systemd → агент Jenkins
```

Ansible-core и коллекции установлены на управляющей VM. Для обычных Python-модулей управляемой VM нужны SSH-доступ и подходящий Python. Постоянный агент Ansible на ней в курсе не устанавливался.

## 4. Установка Ansible и Python-окружение

Сначала создали пользователя ansible, настроили SSH-ключ с Mac и проверили whoami, группы и sudo whoami. Затем установили pipx и Git через APT.

```bash
sudo apt update
sudo apt install pipx git
pipx install 'ansible-core>=2.21,<2.22'
pipx ensurepath
export PATH="$HOME/.local/bin:$PATH"
```

Ansible-core содержит механизм выполнения playbook и встроенные модули. Дополнительные коллекции подключались отдельно, когда понадобились Docker Compose и управление authorized_keys.

Pipx создал отдельное Python-окружение. Команды появились в `/home/ansible/.local/bin`. Установка выполнялась под пользователем ansible без sudo, чтобы окружение принадлежало ему.

В подтверждённом выводе были:

```text
ansible-playbook [core 2.21.6]
Python 3.14.4
/home/ansible/.local/share/pipx/venvs/ansible-core/
```

При первоначальной установке config file имел значение None. Позже появился проектный ansible.cfg.

Первая проверка выполнялась локально на управляющей VM:

```bash
ansible localhost -i 'localhost,' -c local \
  -m ansible.builtin.ping \
  -e "ansible_python_interpreter=$(command -v python3)"
```

Запятая в `localhost,` обозначает inventory-список прямо в аргументе. `-c local` выбирает локальное выполнение. Это подтвердило, что Ansible установлен и может выполнить Python-модуль без SSH.

## 5. SSH и цепочка доверия

### Какие подключения различались

| Подключение | Ключ или механизм |
|---|---|
| Mac → VM Ansible | Личный ключ пользователя Mac |
| VM Ansible → VM Jenkins | `/home/ansible/.ssh/ansible_learning` |
| VM Ansible → GitHub | `/home/ansible/.ssh/github_ansible_practice` |

Ключи находятся на разных машинах и решают разные задачи. Успешный вход с Mac не означает, что VM Ansible уже имеет нужный ключ.

Ключ Ansible создали командой ssh-keygen с ED25519 и парольной фразой. Публичную часть `.pub` добавили в authorized_keys root на новой VM Jenkins.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_learning -C 'ansible-learning'
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/ansible_learning
```

SSH-agent хранит разблокированный ключ в памяти и позволяет использовать его без повторного ввода парольной фразы при каждом подключении. Текущая оболочка должна знать сокет агента через переменные окружения. Новая SSH-сессия не обязательно получает эту связь.

### Почему обычный ssh запрашивал пароль

Команда с явным `-i ~/.ssh/ansible_learning` сработала. Команда без `-i` предлагала пароль root. Имя файла ansible_learning произвольное, и SSH не обязан автоматически выбирать его как стандартный ключ.

На Mac подходящий ключ выбирался автоматически через стандартное имя, SSH-agent или конфигурацию. Конкретный механизм на Mac отдельно не подтверждали.

Различали три запроса:

- Парольная фраза ключа разблокирует локальный закрытый ключ.
- Пароль root/devops подтверждает вход или sudo на удалённой машине.
- Пароль Vault расшифровывает файл переменных Ansible.

### Проверка без парольного fallback

```bash
ssh -i ~/.ssh/ansible_learning \
  -o IdentitiesOnly=yes -o BatchMode=yes \
  root@179.236.42.247 'whoami && hostname'
```

Получили root и nsk-1-vm-7wq8. BatchMode запрещает интерактивный запрос пароля сервера. Успех этой команды подтвердил рабочую аутентификацию ключом.

Ключ хоста SSH проверяется отдельно от пользовательского ключа. При первоначальном входе сверяли отпечаток с серверной консолью. Отключение проверки ключей хоста в проект не добавляли.

## 6. Inventory и выбор серверов

Inventory описывает управляемые узлы, группы и параметры подключения. Имя inventory.ini выбрано нами. Формат INI использует квадратные скобки для группы и строки узлов под ней.

Текущий файл:

```ini
[jenkins]
jenkins_server ansible_host=179.236.42.247 ansible_user=devops ansible_ssh_private_key_file=/home/ansible/.ssh/ansible_learning ansible_python_interpreter=/usr/bin/python3
```

Значения имеют разный смысл:

| Элемент | Значение |
|---|---|
| jenkins | Имя группы Ansible |
| jenkins_server | Придуманное нами имя узла в inventory |
| ansible_host | Реальный IP/DNS для SSH |
| ansible_user | Пользователь на управляемой VM |
| ansible_ssh_private_key_file | Файл ключа на управляющем узле |
| ansible_python_interpreter | Python на управляемом узле |

Имя jenkins_server не меняет hostname Ubuntu. Его можно было назвать иначе, сохранив тот же IP. Это адресуемое имя записи Ansible.

В команде `ansible jenkins ...` слово jenkins является шаблоном выбора узлов. В нашем случае он совпадает с группой. Можно выбрать один узел через jenkins_server или все узлы inventory через all.

```bash
ansible jenkins -i inventory.ini -m ansible.builtin.ping
ansible jenkins_server -i inventory.ini -m ansible.builtin.ping
```

Обе команды обращались к одному серверу, поскольку в группе был один узел.

## 7. Первый удалённый модуль

Команда:

```bash
ansible jenkins -i inventory.ini -m ansible.builtin.ping
```

разбирается следующим образом. Ansible читает inventory, выбирает группу, получает адрес, пользователя, ключ и Python. Затем устанавливает SSH-соединение, подготавливает выполнение модуля на удалённой машине и собирает результат.

```text
Управляющая VM
  читает inventory
  выбирает jenkins_server
  открывает SSH
       → VM Jenkins выполняет Python-модуль
       ← возвращает структурированный результат
  печатает SUCCESS
```

Получили:

```json
{
  "changed": false,
  "ping": "pong"
}
```

`ansible.builtin.ping` проверяет возможность выполнить Python-код через используемое подключение. Это другая проверка, чем ICMP ping. Успех подтвердил больше, чем сетевую доступность IP.

Встроенный модуль имеет полное имя ansible.builtin.ping. Первая часть указывает коллекцию, последняя — модуль. Такие полные имена помогают однозначно понимать, какой код используется.

## 8. Playbook и формат YAML

Разовая команда задаёт одну операцию. Playbook сохраняет последовательность операций в файле. Он содержит список play. Каждый play выбирает hosts, настройки выполнения и список tasks.

Первый пример:

```yaml
- name: Prepare learning directory
  hosts: jenkins
  gather_facts: false

  tasks:
    - name: Ensure learning directory exists
      ansible.builtin.file:
        path: /opt/ansible-learning
        state: directory
        owner: root
        group: root
        mode: '0755'
```

Внешний `- name` начинает элемент списка play. `hosts` выбирает группу inventory. В tasks каждый `- name` начинает задачу. Имя задачи предназначено для человека, а модуль и параметры определяют действие.

YAML использует отступы для вложенности. Важно сохранять пробелы и уровни. Права файлов заключали в кавычки, например '0755', чтобы представить их как строку ожидаемого формата.

Проверка синтаксиса:

```bash
ansible-playbook -i inventory.ini first-playbook.yml --syntax-check
```

Выполнение:

```bash
ansible-playbook -i inventory.ini first-playbook.yml
```

В первых упражнениях inventory подключался как root. После перехода на devops административным playbook потребовались become и запрос sudo-пароля.

## 9. Желаемое состояние и идемпотентность

Модулю file сообщили, каким должен быть каталог. Если каталог отсутствует, он создаётся. Если свойства отличаются, модуль исправляет их. Если состояние уже подходит, изменений нет.

Первое присланное выполнение уже показало `ok=1 changed=0`. Это означало, что каталог с нужными свойствами существовал к моменту проверки. Чтобы увидеть создание отдельно, предложили другой путь /opt/ansible-learning-second.

Идемпотентность проверяли повторным запуском. При неизменном желаемом и фактическом состоянии повторный запуск не должен вносить лишние изменения.

Результаты задач и итог PLAY RECAP:

| Поле | Значение |
|---|---|
| ok | Успешные задачи, включая изменившие состояние |
| changed | Успешные задачи, сообщившие об изменении |
| unreachable | Узлы, к которым не удалось обратиться |
| failed | Ошибки выполнения |
| skipped | Задачи, пропущенные по условиям или механизму выполнения |
| rescued | Ошибки, обработанные rescue-блоками |
| ignored | Ошибки, явно проигнорированные настройкой |

Наличие changed не является ошибкой. Отсутствие changed не является самостоятельным доказательством работоспособности приложения. Поэтому позже добавили проверку через API Jenkins.

У модуля command есть особенность. Он запускает произвольную программу и по умолчанию сообщает changed, даже если команда вроде whoami только читает сведения. Для таких проверок явно задавали changed_when: false.

## 10. Изменения файлов и check mode

К каталогу добавили hello.txt через copy с параметром content. YAML-символ `|` сохраняет многострочный текст.

```yaml
- name: Write learning file
  ansible.builtin.copy:
    dest: /opt/ansible-learning/hello.txt
    content: |
      This server is managed by Ansible.
      This file was created during our course.
    owner: root
    group: root
    mode: '0644'
```

Copy управляет целевым содержимым файла. На сервере прочитали обе строки. Затем вручную заменили текст на Changed manually и повторно выполнили playbook. Ansible восстановил описанное содержимое.

Так изучили расхождение между описанием и сервером. Ansible устраняет его при запуске. Постоянного процесса, отслеживающего этот файл между запусками, у нас нет.

`--check` позволяет поддерживающим его модулям предсказать изменения. `--diff` выводит различия. Сначала запускали проверку, затем применяли и повторяли.

Check mode зависит от модуля и доступного состояния. На чистом сервере задача, требующая каталога из предыдущей неприменённой задачи, может не дать полноценного прогноза. Произвольная command-задача без специальных настроек тоже может быть пропущена. Полный проект не объявлялся проверенным в check mode для всех сценариев.

Diff файлов может содержать секреты. Для файла agent.secret позднее использовали diff: false и no_log: true.

## 11. Переменные и подстановка значений

Вынесли повторяющийся путь и сообщение в vars:

```yaml
vars:
  learning_dir: /opt/ansible-learning
  learning_message: This server is managed by Ansible.
```

Задачи использовали выражения:

```yaml
path: "{{ learning_dir }}"
dest: "{{ learning_dir }}/hello.txt"
```

Это подстановка Jinja2. Когда YAML-значение начинается с `{{`, заключали строку в кавычки. Получившееся значение отправляется модулю.

Проверили переопределение на один запуск:

```bash
ansible-playbook -i inventory.ini first-playbook.yml \
  -e '{"learning_message":"Hello from an extra variable"}' --diff
```

Значение extra vars имеет высокий приоритет. Оно изменило файл на сервере, но не переписало исходный YAML. Обычный запуск вернул сообщение из конфигурации проекта.

Этот же принцип позже применили для проверки пустого секрета. Команда с `-e` намеренно передавала пустое значение, не изменяя зашифрованный файл.

## 12. Group vars и связь имён

Следующим шагом переменные группы вынесли в group_vars/jenkins.yml. Позже файл преобразовали в каталог group_vars/jenkins/ с main.yml и vault.yml.

```text
inventory.ini
  [jenkins]
    jenkins_server
          ↑ получает переменные группы
  group_vars/jenkins/main.yml
  group_vars/jenkins/vault.yml
```

Имя jenkins в group_vars соответствует группе inventory. Имя playbook не определяет эту связь. Задачи получают переменные выбранного узла, в том числе переменные всех подходящих групп.

Текущие обычные переменные группы:

```yaml
learning_dir: /opt/ansible-learning
learning_message: This server is managed by Ansible.
jenkins_api_user: devops
```

Последнее значение — пользователь интерфейса Jenkins. Совпадение с именем Ubuntu-пользователя является нашим выбором, это разные учётные записи.

Для просмотра inventory использовали:

```bash
ansible-inventory -i inventory.ini --host jenkins_server
```

После появления Vault эта команда тоже может требовать пароль. Если она показывает расшифрованные групповые переменные, в вывод могут попасть секреты. Такой полный вывод не следует передавать в чат. Defaults роли загружаются при подключении роли и не обязаны отображаться как переменные inventory.

## 13. Facts и условия

В inspect-server.yml включили gather_facts: true. Ansible сначала выполнил Gathering Facts, затем debug вывел hostname, Ubuntu, архитектуру и память.

Подтверждённые данные:

```text
Hostname: nsk-1-vm-7wq8
OS: Ubuntu 26.04
Architecture: x86_64
Memory: 1895 MiB
```

Facts доступны в словаре ansible_facts. Выражение ansible_facts['hostname'] выбирает значение по ключу. inventory_hostname, напротив, является именем узла из inventory — jenkins_server.

Условие when для Ubuntu выполнилось, для Rocky Linux показало skipping:

```yaml
when: ansible_facts['distribution'] == 'Ubuntu'
```

В when выражение пишется без внешних `{{ }}`. Ansible уже ожидает условие.

Затем в реальной подготовке использовали assert. When пропускает задачу, если условие ложно. Assert останавливает play с ошибкой, если обязательное условие не выполнено. Для данного проекта проверяется Ubuntu. Шаблон Docker рассчитан на amd64, но отдельного assert архитектуры в текущем коде нет.

## 14. Пакеты и повышение прав

Модуль apt обеспечивает наличие пакетов:

```yaml
ansible.builtin.apt:
  name: tree
  state: present
  update_cache: true
  cache_valid_time: 3600
```

Present требует установленный пакет. Это отличается от обязательного обновления до последней версии. Cache_valid_time позволяет переиспользовать свежий кеш APT в течение часа.

Tree установили и проверили его версию на удалённой VM. Позднее name передавали список пакетов. Для apt отдельный loop для каждого пакета не понадобился.

Создали пользователя devops, добавили его в sudo, установили ключ и вручную задали пароль командой passwd. После этого inventory переключили с root на devops.

Разница параметров:

| Настройка | Что определяет |
|---|---|
| ansible_user | Пользователь SSH-входа |
| become: true или -b | Повышение прав для выполнения задачи |
| -K / --ask-become-pass | Запрос пароля механизма повышения прав |
| --ask-vault-pass | Запрос пароля расшифровки переменных |

Whoami без become вернул devops, с become — root. Затем become: true добавили на уровне play установки пакетов.

Become не устанавливает sudo и не выдаёт права пользователю само по себе. Оно использует уже настроенный механизм. В новой VM эту подготовку обеспечивает bootstrap.

## 15. Handlers и порядок выполнения

В демонстрации запись demo.conf через copy отправляла notify обработчику Report configuration change. При changed обработчик выводил сообщение. Повторный запуск с тем же файлом не запускал его. Изменение revision вызвало handler снова.

```yaml
notify: Report configuration change
```

именно по имени связывается с обработчиком в handlers. Уведомления обычно исполняются после обычных задач соответствующего этапа play. Несколько уведомлений одного обработчика до выполнения объединяются.

В Docker-роли handler обновляет APT-кеш при изменении ключа или источника. Meta flush_handlers выполняет ожидающие обработчики до установки пакетов, иначе apt install мог бы начаться раньше обновления списка нового репозитория.

В роли агента handler выполняет daemon_reload и restart. Его уведомляют изменения agent.jar, agent.secret и unit-файла. Перед API-проверкой снова используется flush_handlers, чтобы проверять уже актуальный процесс.

Flush_handlers запускает все ожидающие handlers текущего контекста play, а не только обработчик с похожим именем. Отдельное имя роли само по себе не превращает его в локальную изолированную очередь.

При ошибке до обычного выполнения handlers их применение может не произойти. Успех записи файла ещё не доказывает, что процесс уже перечитал его. Поэтому проверки расположены после явного flush.

## 16. Шаблоны Jinja2

Content удобно для короткого текста. Более длинную конфигурацию вынесли в templates/demo.conf.j2:

```jinja2
# Managed by Ansible
server_name={{ inventory_hostname }}
learning_dir={{ learning_dir }}
message={{ learning_message }}
```

Модуль template читает src на управляющей VM, подставляет переменные выбранного узла и управляет dest на удалённой VM. Проверенный результат содержал server_name=jenkins_server и конкретные значения пути и сообщения.

Расширение j2 является соглашением об имени шаблона. Главный механизм — использование template и выражений Jinja2.

В ролях template ищет файлы в templates соответствующей роли. Поэтому после переноса шаблона в roles/jenkins/templates в src оставили только jenkins-compose.yaml.j2.

Результирующие файлы управляются Ansible. Изменение их вручную может быть перезаписано следующим запуском. Постоянные изменения следует вносить в шаблон или переменные.

## 17. Роли и окончательная структура проекта

Роль объединяет задачи и ресурсы одной части системы. Сначала из общего файла выделили jenkins, затем docker и jenkins_agent. В завершение системные задачи перенесли в server_base.

```text
ansible-practice/
  ansible.cfg
  inventory.ini
  requirements.yml
  bootstrap.yml
  prepare-jenkins.yml
  reboot-jenkins.yml
  create-admin.yml
  first-playbook.yml
  inspect-server.yml
  install-tools.yml
  handlers-demo.yml
  group_vars/
    jenkins/
      main.yml
      vault.yml
  templates/
    demo.conf.j2
  roles/
    server_base/
      tasks/main.yml
    docker/
      tasks/main.yml
      handlers/main.yml
      templates/docker.sources.j2
    jenkins/
      defaults/main.yml
      tasks/main.yml
      templates/jenkins-compose.yaml.j2
    jenkins_agent/
      defaults/main.yml
      tasks/main.yml
      handlers/main.yml
      templates/jenkins-agent.service.j2
  COURSE_REPORT.md
```

Tasks/main.yml содержит список задач, поэтому в нём нет внешних hosts и tasks. Defaults/main.yml содержит стандартные значения. Handlers/main.yml содержит обработчики. Templates содержит исходники конфигурационных файлов.

Финальный основной playbook:

```yaml
- name: Prepare Jenkins server
  hosts: jenkins
  become: true
  gather_facts: true

  tasks:
    - name: Prepare operating system
      ansible.builtin.import_role:
        name: server_base
      tags:
        - server_base

    - name: Configure Docker
      ansible.builtin.import_role:
        name: docker
      tags:
        - docker

    - name: Configure Jenkins
      ansible.builtin.import_role:
        name: jenkins
      tags:
        - jenkins

    - name: Prepare Jenkins agent
      ansible.builtin.import_role:
        name: jenkins_agent
      tags:
        - jenkins_agent
```

Import_role подключает задачи роли в выбранном месте. Так явно сохраняется порядок зависимостей. Мы не настраивали автоматические зависимости ролей через meta/main.yml.

Имя роли jenkins и имя группы jenkins имеют разное назначение. Совпадение удобно для чтения, но выбор серверов происходит по hosts, а выбор каталога роли — по import_role.name.

Для изученных источников переменных использовали порядок от меньшего приоритета к большему — defaults роли, group_vars, vars в play, extra vars через -e. Полная система приоритетов Ansible шире этой учебной схемы.

## 18. Роль server_base

Эта роль готовит общий фундамент VM Jenkins.

### Проверка ОС и пакеты

Assert требует Ubuntu. Apt устанавливает ca-certificates, curl и Git с обновлением кеша по необходимости. Затем устанавливается openjdk-21-jre-headless для агента на хосте.

Контроллер Jenkins имеет свою Java в контейнере. Java на Ubuntu нужна отдельному процессу агента. Это два разных окружения.

### Проверка Java

Command выполняет `/usr/bin/java -version`, register сохраняет результат в java_version, changed_when: false сообщает об отсутствии изменений. Debug читает stderr_lines, поскольку java -version пишет версию в stderr даже при успехе.

Получили Java 21.0.12.1. Наличие текста в stderr не означает неуспех команды, код завершения проверяется отдельно.

### Подготовка swap

На прежней VM Jenkins компиляция Go вызывала OOM. Поэтому на новом сервере сразу подготовили swap 2 ГиБ.

Текущая роль делает следующие действия:

1. Fallocate создаёт /swapfile размером 2G с creates: /swapfile.
2. File задаёт владельца root и mode 0600.
3. Stat получает сведения, debug выводит размер.
4. Blkid читает сигнатуру формата.
5. Assert допускает только пустой тип или swap.
6. Mkswap выполняется только для пустого типа.
7. Swapon --show получает список активных swap-файлов.
8. Swapon /swapfile выполняется только при отсутствии пути в списке.
9. Lineinfile обеспечивает строку в /etc/fstab.

Размер проверенного файла — 2147483648 байт. Creates предотвращает повторный вызов fallocate при существовании файла, но не исправляет его размер. Stat в текущей роли выводит размер, отдельного assert размера нет.

Blkid допускает rc 0 и 2 через failed_when. Неожиданная распознанная файловая система останавливает форматирование. При нестандартных повреждениях существующего файла автоматическое форматирование следует оценивать отдельно.

Строка fstab:

```text
/swapfile none swap sw 0 0
```

Настройка в fstab отвечает за последующие загрузки. Swapon отвечает за текущее состояние. Эти действия проверяются отдельно.

При повторном запуске mkswap и swapon пропускались. Swap помогает переживать пики памяти с использованием диска, но не увеличивает физическую RAM и может замедлять работу под нагрузкой.

## 19. Роль docker

### Описание репозитория

Сначала создаётся `/etc/apt/keyrings`, затем get_url скачивает официальный ключ Docker в docker.asc. Template создаёт `/etc/apt/sources.list.d/docker.sources`.

```jinja2
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: {{ ansible_facts['distribution_release'] }}
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
```

Distribution_release берётся из facts. В нашем случае это resolute. Архитектура amd64 пока задана явно.

При изменении ключа или шаблона отправляется уведомление Refresh Docker APT cache. Затем flush_handlers обеспечивает актуальный кеш до установки пакетов.

### Установка и служба

Apt устанавливает docker-ce, docker-ce-cli, containerd.io, docker-buildx-plugin и docker-compose-plugin. Systemd_service обеспечивает state: started и enabled: true.

Started требует работу сейчас и не делает безусловного restart. Enabled требует запуск при загрузке. Перезапуск является отдельным действием.

Проверяли Docker Client/Server, версию Compose, active и enabled службы, затем hello-world.

Ansible получает право через become на управляемой VM. Устанавливать Docker на саму VM Ansible ради управления этим сервером не потребовалось.

## 20. Роль jenkins

Defaults роли:

```yaml
jenkins_dir: /opt/jenkins
jenkins_image: jenkins/jenkins:lts-jdk21
```

Сначала эти значения находились в group_vars, затем их перенесли в defaults роли. Групповые переменные могут переопределить стандартное значение.

Задачи создают каталог, формируют Compose из шаблона и вызывают community.docker.docker_compose_v2.

```yaml
community.docker.docker_compose_v2:
  project_src: "{{ jenkins_dir }}"
  state: present
  pull: missing
```

Модуль использует Compose CLI на VM Jenkins. Он находится в коллекции community.docker, которая не входит в ansible-core. Путь project_src указывает на управляемую VM.

Present применяет проект. Pull missing скачивает отсутствующий образ, но не требует скачивать обновление существующего локального тега при каждом запуске.

Шаблон Compose:

```yaml
name: jenkins-learning

services:
  jenkins:
    image: "{{ jenkins_image }}"
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8080"
    volumes:
      - jenkins_home:/var/jenkins_home

volumes:
  jenkins_home:
```

Явное имя проекта сохраняет стабильное имя тома jenkins-learning_jenkins_home. Файлы Compose принадлежат root. Внутри тома Jenkins хранит пользователей, настройки, задания, плагины и другие данные.

После первого запуска вручную прошли Unlock Jenkins, suggested plugins и создание администратора. Начальный пароль прочитали через Docker exec. Позднее вручную добавляли узел и API-токен. Эти операции в роль пока не включены.

При записи нового Compose-файла handler не понадобился, поскольку следующая обычная задача сразу применяет текущую конфигурацию проекта.

## 21. Роль jenkins_agent

### Стандартные значения

```yaml
jenkins_agent_user: jenkins-agent
jenkins_agent_dir: /home/jenkins-agent/agent
jenkins_controller_url: http://127.0.0.1:8080
jenkins_agent_name: docker-builder
```

URL loopback относится к VM, на которой выполняется модуль. В нашем варианте агент и контейнер контроллера находятся на одном хосте. Перенос агента на другую VM потребовал бы изменения URL и сети.

### Подготовка пользователя и файлов

Роль проверяет обязательные переменные, создаёт jenkins-agent с группой docker, создаёт каталоги, скачивает agent.jar, пишет секрет и unit-файл, включает службу и проверяет online через API.

Группа docker позволяет пользователю агента обращаться к системному Docker Engine. Это сильное административное право на VM, оно выбрано для выделенного сервера сборки.

Рабочий каталог имеет mode 0750, tmp — 0700. Файл agent.jar принадлежит агенту и доступен для чтения. Secret имеет mode 0600 и также принадлежит агенту.

Jar скачивается с текущего контроллера по `/jnlpJars/agent.jar`. Задача повторяется при неуспехе. Это позволяет дождаться HTTP-готовности Jenkins после старта контейнера.

### Шаблон systemd

```ini
[Unit]
Description=Jenkins build agent
Wants=network-online.target docker.service
After=network-online.target docker.service

[Service]
User={{ jenkins_agent_user }}
Group={{ jenkins_agent_user }}
WorkingDirectory={{ jenkins_agent_dir }}
ExecStart=/usr/bin/java -Djava.io.tmpdir={{ jenkins_agent_dir }}/tmp -jar {{ jenkins_agent_dir }}/agent.jar -url {{ jenkins_controller_url }}/ -name {{ jenkins_agent_name }} -secret @{{ jenkins_agent_dir }}/agent.secret -webSocket -workDir {{ jenkins_agent_dir }}
Restart=always
RestartSec=10
UMask=0077

[Install]
WantedBy=multi-user.target
```

Java-программа запускается на хосте через systemd. Docker запускает контроллер отдельно. Секрет передаётся Java через файл, а не помещается открытым значением в unit.

Параметр java.io.tmpdir сразу направляет временные файлы на основной диск. В предыдущем курсе системный tmpfs /tmp был меньше порога Jenkins, поэтому агент помечался offline.

After задаёт порядок запуска служб, но не гарантирует, что Jenkins уже готов. Агент повторяет подключение самостоятельно, а playbook дополнительно ждёт готовности API.

### Когда выполняется restart

Get_url agent.jar, copy секрета и template unit-файла уведомляют Restart Jenkins agent. Handler выполняет daemon_reload и state: restarted.

Обычная задача systemd_service использует state: started и enabled: true. Перед проверкой API выполняется flush_handlers.

Повторное применение без изменений не запускало handler. Пользователь сравнил MainPID и время запуска службы до и после playbook и подтвердил сохранение процесса.

## 22. Ansible Vault и разные виды секретов

### Устройство файлов

Переменные группы разделили на два файла:

```text
group_vars/jenkins/main.yml   — обычные настройки
group_vars/jenkins/vault.yml  — зашифрованные значения
```

Файл vault.yml отслеживается Git. Его заголовок начинается с `$ANSIBLE_VAULT;`. Содержимое шифротекста в отчёт не включено.

Имена секретных переменных, добавленных в упражнениях:

| Переменная | Назначение |
|---|---|
| jenkins_agent_secret | Разрешает агенту подключиться как docker-builder |
| jenkins_api_token | Авторизует HTTP-запросы от имени пользователя Jenkins |
| bootstrap_admin_password_hash | Хеш пароля нового devops для bootstrap |

Для шифрования используется пароль Vault. Он хранится отдельно и не восстанавливается из зашифрованного файла.

### Команды Vault

```bash
EDITOR=nano ansible-vault create group_vars/jenkins/vault.yml
EDITOR=nano ansible-vault edit group_vars/jenkins/vault.yml
ansible-vault view group_vars/jenkins/vault.yml
```

View выводит расшифрованные значения, поэтому использовался только локально. Для проверки правильности пароля без вывода содержимого применяли:

```bash
ansible-vault view group_vars/jenkins/vault.yml >/dev/null
```

Playbook запускается с --ask-vault-pass. Шифрование файла не требует передавать пароль через -e или сохранять его в Git.

### Границы защиты

Vault защищает файл в состоянии хранения. Во время выполнения Ansible расшифровывает переменные и использует их. На целевой VM агенту нужен читаемый файл agent.secret, защищённый владельцем и mode 0600.

No_log скрывает результат задачи в обычном выводе, diff: false запрещает показывать содержимое файла. Маскирование не заменяет корректное хранение файлов и ограничение доступа к управляющей машине.

В группе jenkins Vault загружается даже для части операций, которые сами не используют конкретный секрет. Поэтому ping, list-tasks и syntax-check для этого inventory могут требовать пароль Vault.

### Четыре пароля/секрета, которые нельзя путать

1. Парольная фраза SSH-ключа разблокирует ключ на VM Ansible.
2. Become password — пароль devops на VM Jenkins для sudo.
3. Vault password — пароль расшифровки файла на управляющей VM.
4. Секрет агента и API-токен являются значениями внутри Vault с разными назначениями.

При запуске с -K и --ask-vault-pass Ansible может спросить оба пароля. Их порядок в терминале не делает их взаимозаменяемыми.

## 23. Проверка агента через Jenkins API

Работающий Java-процесс может ещё не подключиться к Jenkins или быть offline для новых заданий. Поэтому проверку дополнили запросом состояния узла.

В UI Jenkins создали API-токен пользователя. Имя пользователя хранится как jenkins_api_user в main.yml, токен — в Vault. Токен наследует права своего пользователя. В учебном стенде использовалась существующая административная учётная запись.

Задача uri обращается к:

```text
http://127.0.0.1:8080/computer/docker-builder/api/json?tree=offline
```

В роли имя кодируется фильтром urlencode и берётся из переменной. Запрос выполняется на VM Jenkins по локальному HTTP.

Проверка требует одновременно status 200 и json.offline false. Force_basic_auth отправляет имя пользователя и токен для HTTP Basic. Пароль UI Jenkins в переменную токена не подставляли.

Register сохраняет ответ, until задаёт критерий успеха. Default(0) и default(true) позволяют оценивать ожидание даже при неполном ответе. Bool преобразует значение в логическое представление.

У всей задачи no_log: true. После успеха отдельный debug сообщает, что агент online. Проверка читает состояние, не включает вручную отключённый узел автоматически.

Эта проверка сильнее systemctl active, но не является тестом сборки приложения. Она подтверждает доступность агента для планирования заданий в момент запроса.

## 24. Register, until, assert и loop

### Register

Register присваивает результат задачи переменной. У command обычно есть rc, stdout, stderr и варианты *_lines. У stat есть словарь stat. У uri JSON-ответ доступен как json, когда модуль распознал соответствующий тип содержимого.

Register сохраняет данные текущего запуска. Это не запись значения обратно в inventory и не постоянное хранилище между запусками.

### Until

Until проверяет результат выполненной попытки. При ложном условии используются retries и delay. В проекте так скачивается agent.jar и ожидается online.

Ожидание само по себе не исправляет неверный токен или ошибочный URL. После исчерпания попыток задача завершится ошибкой.

### Assert и предварительная проверка

В начале роли агента проверяются наличие, тип и непустое значение API username, секрета агента и API-токена. Для строк применяется trim перед length.

Тест ошибки:

```bash
ansible-playbook prepare-jenkins.yml --tags jenkins_agent \
  -K --ask-vault-pass -e '{"jenkins_agent_secret":""}'
```

Extra vars переопределили значение только на один запуск. Assert остановил роль до изменения пользователя, файла секрета или службы. Следующий обычный запуск снова использовал настоящий Vault.

Предварительная проверка расположена в начале роли агента. При полном запуске более ранние роли server_base/docker/jenkins уже могли выполнить свои изменения. Это локальный порядок, не глобальная транзакция всего playbook.

### Loop

Две задачи создания каталогов объединили:

```yaml
- name: Ensure Jenkins agent directories exist
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: directory
    owner: "{{ jenkins_agent_user }}"
    group: "{{ jenkins_agent_user }}"
    mode: "{{ item.mode }}"
  loop:
    - path: "{{ jenkins_agent_dir }}"
      mode: '0750'
    - path: "{{ jenkins_agent_dir }}/tmp"
      mode: '0700'
  loop_control:
    label: "{{ item.path }}"
```

На каждой итерации item является очередным словарём. Dot notation выбирает его path и mode. Label влияет на отображение результата, но не скрывает секретные значения как no_log.

Loop удобен для однотипных объектов с разными параметрами. Для apt использовали встроенную поддержку списка пакетов, сохранив одну операцию управления пакетами.

## 25. Tags и ansible.cfg

Tags добавили к import_role. При статическом импорте метки доступны задачам роли. Установлены server_base, docker, jenkins, jenkins_agent.

```bash
ansible-playbook prepare-jenkins.yml --list-tags --ask-vault-pass
ansible-playbook prepare-jenkins.yml --tags jenkins_agent --list-tasks --ask-vault-pass
ansible-playbook prepare-jenkins.yml --tags jenkins_agent -K --ask-vault-pass
```

Выбор tags не разрешает зависимости автоматически. На чистом сервере запуск только jenkins_agent не установит Java, Docker и контроллер. На подготовленном сервере это удобно для обновления агента.

Gathering Facts может выполняться и при таком фильтре. Поэтому число задач в recap не обязано совпадать с числом строк основного playbook.

Файл ansible.cfg:

```ini
[defaults]
inventory = ./inventory.ini
roles_path = ./roles
```

Он задаёт стандартный inventory и путь ролей для проекта. После его создания команды перестали требовать -i inventory.ini. Ansible --version показывал абсолютный путь найденного файла конфигурации.

Запускать проект договорились из его корня. При другом текущем каталоге Ansible может найти другой конфигурационный файл. При итоговом клонировании отдельно проверили config file новой копии.

В присланном перед последним рефакторингом файле была запись `-jenkins_agent` без пробела. Финальный prepare-jenkins.yml содержит корректный список с `- jenkins_agent`.

## 26. Управление SSH-ключами

Первоначальный create-admin.yml использовал copy для authorized_keys. Copy задавал всё содержимое файла и мог удалить ключи, которые добавили отдельно.

Заменили его модулем ansible.posix.authorized_key:

```yaml
ansible.posix.authorized_key:
  user: devops
  key: "{{ lookup('ansible.builtin.file', '/home/ansible/.ssh/ansible_learning.pub') }}"
  state: present
  exclusive: false
```

Lookup file читает публичный ключ на управляющем узле. Модуль изменяет разрешённые ключи пользователя на целевой VM. Exclusive false сохраняет остальные записи.

Для модуля установили коллекцию ansible.posix. В текущем requirements.yml у неё версия не указана. В упражнении предлагалось записать установленную версию, но в проверенном коммите это не сделано. Community.docker закреплена как 5.4.0.

Create-admin.yml остался в проекте как отдельное упражнение и средство управления существующим администратором. Bootstrap использует тот же подход к ключу, но дополнительно умеет создать нового пользователя с паролем.

## 27. Bootstrap и обычная настройка

### Почему два входа

Основной inventory подключается как devops. На чистой VM такого пользователя ещё нет. Нужен первый проход под уже доступным root, который создаст административный вход.

Bootstrap.yml использует hosts jenkins, become false и play vars ansible_user: root. Эта переменная имеет приоритет над ansible_user из inventory. Путь ключа и адрес узла продолжают браться из inventory.

Become false выбран потому, что SSH-вход уже root. Для обычного prepare-jenkins.yml используется devops и become true.

### Подготовка пароля

На управляющем узле выполнили:

```bash
openssl passwd -6
```

Команда интерактивно получила пароль и вывела SHA-512 crypt-хеш с префиксом $6$. Его записали в bootstrap_admin_password_hash внутри Vault. Хеш сохраняется целиком с солью и должен рассматриваться как секретный материал.

В bootstrap проверяется тип и префикс, затем user создаёт devops с shell /bin/bash, домашним каталогом и группой sudo. Параметры password и update_password: on_create задают пароль только при создании пользователя.

На существующей VM это не меняет текущий пароль. При потере пароля уже существующего пользователя повторение bootstrap не является механизмом его сброса.

### Последовательность

```bash
ansible-playbook bootstrap.yml --ask-vault-pass
ansible-playbook prepare-jenkins.yml -K --ask-vault-pass
```

Для bootstrap root уже должен принимать ключ управляющего узла. Ansible не получает первый доступ из ничего. Подготовка root authorized_keys через провайдера или начальный вход остаётся предпосылкой.

Модули bootstrap используют Python. В нашей Ubuntu он был предустановлен. Подготовка серверов без Python через raw в курсе не разбиралась.

При проверке bootstrap на существующем сервере успех подтвердил повторяемость и доступ. Полное создание администратора последним bootstrap на новой чистой VM отдельно не тестировалось.

## 28. Git, deploy key и восстановление проекта

### Репозиторий

Проект хранится в `https://github.com/kinzaz/ansible-practice.git`. В Git добавлены playbook, роли, шаблоны, inventory, обычные и зашифрованные переменные, список коллекций и ansible.cfg.

Gitignore исключает локальные файлы пароля Vault, .ssh, ключи с типовыми расширениями, ansible_learning, временные файлы и резервные копии before-*.

Gitignore является фильтром имён, а не анализатором секретов. Приватный ключ с другим именем всё ещё мог бы попасть в Git при неосторожном добавлении. Уже отслеживаемый файл не перестаёт отслеживаться только из-за нового правила gitignore.

### Отдельный GitHub-ключ

Создали ключ github_ansible_practice с passphrase и добавили публичную часть в GitHub Deploy keys данного репозитория. Для push требуется Allow write access.

Первый push вернул ошибку, что ключ помечен read only. У deploy key не было Edit. Запись удалили и добавили тот же публичный ключ заново с правом записи. Новый закрытый ключ генерировать не потребовалось.

Конфигурация SSH на управляющей VM:

```sshconfig
Host github-ansible-practice
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_ansible_practice
    IdentitiesOnly yes
```

Remote использует этот alias:

```text
git@github-ansible-practice:kinzaz/ansible-practice.git
```

Этот alias существует на настроенной управляющей VM. Он не появляется автоматически в любой новой машине после клонирования репозитория.

### Проверка клонирования

Создали `~/ansible-practice-restore-check` через git clone, проверили его config file, установили коллекции из requirements и запустили полный playbook из новой копии.

Первая попытка остановилась на расшифровке Vault. Такая же ошибка была и у исходного файла. Пользователь подтвердил, что вводил другой пароль. После правильного пароля проверка завершилась успешно.

Это подтвердило, что нужные файлы проекта сохранены в Git. SSH-ключи, уже установленный Ansible, известные ключи хостов и пароль Vault использовались с существующего управляющего сервера. Полное восстановление control node с нуля этим упражнением не проверялось.

## 29. Повторный запуск и перезагрузка

Идемпотентность проверяли по changed=0 и отдельно по процессу агента:

```bash
sudo systemctl show jenkins-agent -p MainPID -p ActiveEnterTimestamp -p NRestarts
```

Пользователь сравнил значения до и после обычного запуска роли. PID сохранился. Это показало, что handler не перезапускал службу без изменения управляемых файлов.

Для перезагрузки создали отдельный reboot-jenkins.yml. Он явно выбирает jenkins и выполняет become.

```yaml
ansible.builtin.reboot:
  connect_timeout: 5
  reboot_timeout: 600
  post_reboot_delay: 10
  test_command: whoami
```

Модуль инициирует рестарт, проверяет смену загрузки и ожидает выполнения тестовой команды. Reboot_timeout относится к этапам проверки модуля, поэтому его не следует воспринимать как абсолютно точный общий таймер всей операции.

После перезагрузки снова выполнили роль агента. Успешный результат подтвердил восстановление Docker, контроллера, systemd-службы и связи с Jenkins в рамках выполненных проверок. SSH-туннель с Mac пришлось открыть заново.

Работающий процесс и готовое приложение — разные уровни. Поэтому проверка закончилась API online, а не одним systemctl active.

## 30. Разобранные ошибки и их причины

### SSH-agent недоступен

Ssh-add показал Could not open a connection to your authentication agent. Текущая оболочка не знала доступный сокет агента. Выполнили eval ssh-agent -s и ssh-add. Это восстановило возможность использовать ключ с passphrase.

### Ключ разрешён, но обычный ssh просит пароль

Явный -i и BatchMode дали успешный root-вход, а команда без -i запросила пароль. Причина была в выборе ключа клиентом. Inventory затем явно закрепил путь к ansible_learning, поэтому Ansible не зависел от угадывания имени.

### Permission denied при первом ключевом входе

Проверили наличие публичного ключа у правильного пользователя на правильной VM, права authorized_keys и сопоставление отпечатков. После исправления команда без парольного fallback выполнилась успешно.

### Путаница группы и узла

Jenkins в команде оказался именем группы, jenkins_server — именем записи, а nsk-1-vm-7wq8 — hostname Ubuntu. Разобрали все три уровня и связь через ansible_host.

### Переменные group_vars казались несвязанными

Показали совпадение имени группы [jenkins] и файла/каталога group_vars/jenkins. Playbook использует переменные выбранного узла, имя playbook в автоматической связи не участвует.

### Command показывал CHANGED для whoami

Модуль не знает семантику произвольной команды и по умолчанию считает выполнение изменением. Для read-only проверок в реальных ролях добавили changed_when false.

### Ручные изменения возвращались обратно

Это ожидаемый результат управления файлом через copy/template. В playbook описано нужное содержимое. Ручное изменение не меняет исходное описание и исправляется следующим запуском.

### Плагинам Jenkins не хватало зависимости

После ручной установки suggested plugins Jackson 2 API не загрузился из-за отсутствия snakeyaml-api. Gradle и JJWT также не загрузились из-за зависимости от Jackson. Установили SnakeYAML API через UI и перезапустили контейнер. Проблема исчезла.

Ansible-роль контроллера на тот момент управляла контейнером и Compose. Она не устанавливала список Jenkins-плагинов, поэтому исправление было отдельным ручным действием внутри приложения.

### GitHub deploy key был read only

Авторизация прошла, но push получил отказ. Исправили права записи у deploy key через удаление записи и повторное добавление того же публичного ключа с Allow write access.

### Неправильный пароль Vault

Получили Decryption failed. Проверили исходный файл отдельно через ansible-vault view с перенаправлением вывода. Пользователь подтвердил неправильный ввод пароля. После корректного ввода проект выполнился. Сам файл пересоздавать не потребовалось.

### Слишком общая инструкция по выделению роли

Сначала перенос задач был объяснён недостаточно конкретно. После чтения полного prepare-jenkins.yml подготовили готовые тексты обоих файлов. Финальная структура сохранила задачи и исправила ошибку пробела у метки агента.

## 31. Границы автоматизации и текущие нюансы

### Что автоматизировано

- Подготовка devops через bootstrap при доступном root и Python.
- Базовые пакеты и проверка Ubuntu.
- Java для агента.
- Swap и запись fstab.
- Репозиторий Docker, пакеты и служба.
- Каталог и Compose контроллера.
- Пользователь и каталоги агента.
- Доставка agent.jar, secret и unit-файла.
- Старт, автозапуск и необходимый restart агента.
- Проверка online через Jenkins API.
- Отдельная перезагрузка и ожидание SSH.

### Что остаётся ручным

- Создание облачных VM и их сетевых правил.
- Первоначальная проверка SSH host key и установка ключа root.
- Подготовка самой control VM, pipx и SSH-agent.
- Создание администратора Jenkins через мастер.
- Выбор и установка Jenkins-плагинов.
- Регистрация узла docker-builder в UI.
- Создание API-токена и получение секрета агента.
- Перенос секретов в Vault и сохранение пароля Vault отдельно.
- Создание репозитория и настройка GitHub Deploy key.

Jenkins Configuration as Code обсуждался как расширение, но не внедрялся. Старые задания и Credentials предыдущего контроллера из курса Docker автоматически не восстанавливались.

### Нюансы проверенного коммита

1. В requirements.yml закреплена community.docker 5.4.0, но ansible.posix без версии. Следующая установка может выбрать другой выпуск.
2. В first-playbook.yml отсутствует become true. После смены inventory на devops создание/исправление root-owned файлов под /opt требует добавить become и запускать с -K. Уже существующие файлы и успешная проверка содержимого не гарантируют, что ранний учебный playbook способен исправлять их под текущими правами.
3. Repo-шаблон Docker жёстко задаёт amd64. Проверка ОС не проверяет архитектуру отдельно.
4. State present у пакетов не фиксирует точную версию. Повторная подготовка позже может получить другие версии.
5. Jenkins image использует подвижный lts-jdk21. Pull missing переиспользует локальный образ, поэтому одинаковый Git-коммит на двух VM может использовать разные скачанные ранее образы.
6. Создание swap проверяет существование файла, но не исправляет его размер автоматически.
7. Секреты проверяются на непустые строки. Реальная действительность API-токена проверяется позже запросом, а секрета агента — успешным подключением.
8. No_log скрывает полезные подробности ошибки API. Для диагностики сначала отдельно проверяют URL, пользователя, состояние узла и журнал, сохраняя секреты закрытыми.
9. Включение docker-группы даёт агенту широкие возможности на сервере.
10. Хранение зашифрованного Vault в Git требует отдельного сохранения пароля и контроля доступа к нему.
11. На другой control VM абсолютные пути SSH-ключа и lookup потребуют совпадающей структуры или изменения inventory/playbook.
12. Ansible-ошибка не откатывает автоматически все уже применённые задачи play. Это последовательное управление состоянием. Механизм rollback приложения из предыдущего курса решал другую задачу.
13. Одновременный запуск конфигурации из нескольких копий проекта отдельной блокировкой не защищён. Проект использовался последовательными запусками.
14. На Mac при подготовке отчёта ansible-playbook не был обнаружен. Отчёт проверен как документ и по исходникам, без локального повторного запуска ролей.

Эти пункты описывают границы учебного результата. Они не означают, что полный сценарий на чистой VM уже проверен.

## 32. Практический справочник команд

### На VM Ansible — окружение

```bash
whoami
python3 --version
ansible --version
pipx list
ansible-galaxy collection list
ssh-add -l
```

При отсутствии доступного SSH-agent:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/ansible_learning
ssh-add ~/.ssh/github_ansible_practice
```

Ключ GitHub нужен только для Git-операций. Команды Ansible используют ключ целевого сервера.

### Проверка синтаксиса и плана

Из корня проекта:

```bash
ansible-playbook prepare-jenkins.yml --syntax-check --ask-vault-pass
ansible-playbook prepare-jenkins.yml --list-tasks --ask-vault-pass
ansible-playbook prepare-jenkins.yml --list-tags --ask-vault-pass
```

Syntax-check проверяет структуру и доступность нужных компонентов, но не доказывает успешность удалённых операций. List-tasks показывает последовательность без выполнения.

### Inventory и подключение

```bash
ansible jenkins -m ansible.builtin.ping --ask-vault-pass
ansible jenkins_server -m ansible.builtin.command -a whoami --ask-vault-pass
ansible jenkins_server -m ansible.builtin.command -a whoami -b -K --ask-vault-pass
```

Последние две команды показывают разницу SSH-пользователя и пользователя выполнения после become.

### Применение

```bash
ansible-playbook bootstrap.yml --ask-vault-pass
ansible-playbook prepare-jenkins.yml -K --ask-vault-pass
ansible-playbook prepare-jenkins.yml --tags docker -K --ask-vault-pass
ansible-playbook prepare-jenkins.yml --tags jenkins_agent -K --ask-vault-pass
```

Выбор тегов предназначен для подготовленного окружения. Необходимые зависимости следует учитывать явно.

### Перезагрузка

```bash
ansible-playbook reboot-jenkins.yml -K --ask-vault-pass
```

Команда перезагружает целевую VM. Перед запуском следует учитывать выполняющиеся сборки.

### Проверка на управляемой VM Jenkins

```bash
sudo systemctl status docker --no-pager
sudo systemctl status jenkins-agent --no-pager -l
sudo systemctl show jenkins-agent -p MainPID -p ActiveEnterTimestamp -p NRestarts
sudo journalctl -u jenkins-agent -n 50 --no-pager
sudo docker compose -f /opt/jenkins/compose.yaml ps
sudo docker compose -f /opt/jenkins/compose.yaml logs --tail=50 jenkins
free -h
swapon --show
```

Не выводить agent.secret при передаче диагностических данных в чат.

### Туннель с Mac

```bash
ssh -N -o ExitOnForwardFailure=yes \
  -L 127.0.0.1:19090:127.0.0.1:8080 \
  root@179.236.42.247
```

Открыть браузер на `http://127.0.0.1:19090/`. В этом примере использован уже подтверждённый вход root с Mac. Ключ Ansible находится на control VM и автоматически на Mac не появляется.

### Работа с Vault

```bash
EDITOR=nano ansible-vault edit group_vars/jenkins/vault.yml
ansible-vault view group_vars/jenkins/vault.yml >/dev/null
head -n 1 group_vars/jenkins/vault.yml
```

Head показывает только заголовок формата. View без перенаправления печатает секреты.

### Git

```bash
git status --short
git diff
git diff --cached --stat
git add НУЖНЫЕ_ФАЙЛЫ
git commit -m 'Describe the configuration change'
git push
```

Git diff следует просматривать с учётом содержимого файлов. Зашифрованный vault.yml выглядит как шифротекст, но обычные переменные и новые файлы могут содержать случайно добавленные данные.

## 33. Порядок подготовки новой VM

Этот раздел описывает логический порядок на основе проекта. Отдельная полная репетиция на новой чистой VM после завершения всех ролей не выполнялась.

### Восстановить управляющее окружение

1. Создать управляющую VM с поддерживаемым Python.
2. Создать пользователя ansible и доступ с Mac.
3. Установить pipx, Git и выбранную версию ansible-core.
4. Восстановить приватные ключи из отдельного защищённого места или выпустить новые и установить соответствующие публичные ключи.
5. Настроить доступ к GitHub, клонировать проект и установить коллекции.
6. Иметь пароль Vault. Один clone не восстанавливает этот пароль.

### Подготовить целевой сервер

1. Создать VM Ubuntu amd64 с достаточными ресурсами.
2. Проверить root-доступ, host key и Python.
3. Разрешить публичный ключ управляющего узла для root.
4. Обновить ansible_host в inventory.
5. Выполнить bootstrap.yml с правильным Vault.
6. Проверить вход как devops и sudo с паролем, соответствующим bootstrap-хешу.

### Сначала поднять контроллер

Для ещё не настроенного Jenkins требуется разделить создание инфраструктуры и создание секретов приложения.

```bash
ansible-playbook prepare-jenkins.yml \
  --tags server_base,docker,jenkins -K --ask-vault-pass
```

Это подготавливает системные зависимости и запускает контейнер. В существующем Vault могут быть значения старого Jenkins, поэтому сразу запускать агент с ними на новом пустом контроллере нельзя считать корректным восстановлением.

Затем открыть туннель, пройти первоначальный мастер, установить плагины, создать пользователя и узел docker-builder. Настроить remote root `/home/jenkins-agent/agent`, label docker-builder, один executor и inbound launch.

### Подготовить актуальные секреты

Получить секрет нового узла, создать новый API-токен, обновить Vault, проверить jenkins_api_user. После этого запустить:

```bash
ansible-playbook prepare-jenkins.yml \
  --tags jenkins_agent -K --ask-vault-pass
```

Ожидать online. Если контроллер восстановлен из прежнего тома с прежними настройками, действительность сохранённых секретов всё равно следует проверить.

### Проверить повторяемость

Повторить полный playbook, проверить changed, PID агента и API online. При необходимости выполнить отдельный reboot-playbook, затем снова проверить готовность. Сохранить подтверждённые изменения в Git.

Проверка восстановления из Git в курсе соответствовала повторному применению на уже настроенной VM. Этот раздел расширяет её до понятного плана будущей полной репетиции, сохраняя существующие ручные предпосылки.

## 34. Словарь и вопросы для самопроверки

### Словарь

| Термин | Значение в нашем проекте |
|---|---|
| Control node | VM, с которой запускается Ansible |
| Managed node | VM Jenkins, к которой обращается Ansible |
| Inventory | Список узлов, групп и параметров подключения |
| Host pattern | Выражение выбора узлов, например jenkins |
| Playbook | YAML-файл со списком play |
| Play | Выбор hosts и набор задач с общими настройками |
| Task | Вызов модуля с параметрами |
| Module | Компонент, выполняющий операцию и возвращающий результат |
| Collection | Набор модулей и плагинов, например community.docker |
| Role | Структурированный набор задач, шаблонов и переменных |
| Facts | Сведения, полученные о сервере |
| Variable | Именованное значение конфигурации |
| Defaults | Стандартные значения роли с низким приоритетом |
| Handler | Задача, уведомляемая при изменении |
| Notify | Уведомление handler по имени |
| Template | Файл с Jinja2-выражениями для формирования конфигурации |
| Register | Сохранение результата задачи в переменную |
| Until | Условие повторения задачи до успеха |
| Assert | Проверка обязательного условия |
| Become | Повышение или изменение пользователя выполнения |
| Tag | Метка для выбора части задач |
| Vault | Шифрование переменных/файлов Ansible |
| Idempotence | Повторное применение сохраняет уже достигнутое состояние |
| Bootstrap | Первоначальная подготовка доступа и предпосылок |

### Вопросы

1. На какой машине находится приватный ключ, указанный в inventory?
2. Чем отличаются jenkins, jenkins_server и hostname Ubuntu?
3. Что проверяет ansible.builtin.ping?
4. Чем один вызов ansible отличается от ansible-playbook?
5. Какие элементы YAML образуют список, а какие словарь?
6. Почему права файлов записаны строками в кавычках?
7. Почему changed=0 не доказывает готовность Jenkins к сборкам?
8. Почему command whoami показал CHANGED и как мы это исправили?
9. Как файл group_vars связан с группой inventory?
10. Чем inventory_hostname отличается от facts hostname?
11. Почему в when не пишут внешние фигурные скобки?
12. Чем when отличается от assert?
13. Как become связано с ansible_user и sudo?
14. Чем пароль Vault отличается от BECOME password?
15. Когда handler попадает в очередь и когда выполняется?
16. Зачем flush_handlers стоит перед установкой Docker и перед API-проверкой агента?
17. Где выполняется lookup file, а где модуль copy?
18. Откуда роль получает defaults и кто может их переопределить?
19. Чем src template отличается от dest?
20. Почему Docker Compose-модуль потребовал коллекцию?
21. Чем state present отличается от обновления до последнего пакета?
22. Что защищает creates у fallocate, а чего он не проверяет?
23. Почему нельзя безусловно выполнять mkswap при каждом запуске?
24. Как сохраняется swap после перезагрузки?
25. Что делает item в loop с каталогами?
26. Как role jenkins_agent реагирует на изменение секрета или jar?
27. Почему active-службы недостаточно для проверки агента?
28. Какими правами обладает Jenkins API-токен?
29. Что скрывает no_log, а что защищает Vault?
30. Почему authorized_key предпочтительнее copy для добавления одного ключа?
31. Что даёт exclusive false?
32. Почему --tags jenkins_agent не подходит для первого запуска на чистой VM?
33. Почему bootstrap использует play var ansible_user root?
34. Почему update_password on_create не сбрасывает пароль существующего пользователя?
35. Какие данные из Git не восстановятся одним clone?
36. Почему одинаковый тег Jenkins image не гарантирует одинаковые байты на двух VM?
37. Какие версии коллекций закреплены в текущем requirements?
38. Какие части Jenkins ещё создаются через интерфейс?
39. Каким результатом подтвердилась проверка клонированного проекта?
40. Что ещё требуется проверить для утверждения о полном восстановлении стенда с нуля?

### Ссылки на первоисточники

Материалы ниже дополняют конспект. Фактически выполненные действия и исторические версии описаны в разделах выше.

- [Установка Ansible](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html)
- [Inventory](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html)
- [Playbook](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_intro.html)
- [Переменные](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_variables.html)
- [Roles](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Handlers](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_handlers.html)
- [Ansible Vault](https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html)
- [Docker Compose v2 module](https://docs.ansible.com/projects/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
- [Authorized key module](https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/authorized_key_module.html)
- [Jenkins Remote Access API](https://www.jenkins.io/doc/book/using/remote-access-api/)
