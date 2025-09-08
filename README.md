# Автономная CI/CD инфраструктура с GitLab, Nexus и Docker. Автоматизированно через Ansible

Решение для изолированной сборки Docker-образов с кэшированием зависимостей в условиях ограниченного доступа к
интернету.

## 📌 Особенности

- **Полная изоляция**: Возможна работа без доступа к Docker Hub и внешним репозиториям Go
- **Кэширование артефактов**:
    - Docker-образы через Nexus Proxy (Docker Hub mirror)
    - Golang модули через Go Proxy
- **IaС**: Использование Ansible для автоматизации развёртывания

## 🏗 Архитектура

```plaintext
+----------------+        +-----------------+        +---------------+
|   GitLab CE    |        |  GitLab Runner  |        |   Nexus 3     |
| (Code Hosting) |<------>| (CI/CD Executor)|<------>| (Artifacts &  |
|                |        |                 |        |  Dependencies)|
+----------------+        +-----------------+        +---------------+
```

### Компоненты

1. **GitLab CE** (Docker контейнер):
    - Хостинг репозиториев
    - Управление пайплайнами

2. **GitLab Runner** (Docker контейнер):
    - Выполнение CI/CD задач

3. **Nexus 3** (Docker контейнер):
    - Репозитории:
        - `docker-hosted`
        - `docker-proxy`
        - `golang-proxy`

## ⚙️ Установка

### Предварительные требования

- 3 ВМ с Debian 12
- SSH доступ к ВМ
- Статические IP для каждой ВМ
- Минимум 4 ГБ ОЗУ на каждой ВМ
- Протестировано с Ansible 2.16+

### 1. Клонируйте репозиторий

```bash
git clone git@github.com:Jubastik/Stable-Build.git
cd Stable-Build/ansible
```

### 2. Создайте inventory файл на основе примера

```bash
cp inventory.example inventory
```

### 3. Отредактируйте inventory файл

```bash
nano inventory
```

### 4. Создайте и настройте файлы переменных

Главные настройки расположены в `group_vars/all`

```bash
cd group_vars
cp all.yml.example all.yml
nano all.yml
```

Остальные файлы не содержат критических настроек

```bash
cp nexus.yml.example nexus.yml
nano nexus.yml
```

```bash
cp gitlab.yml.example gitlab.yml
nano gitlab.yml
```

```bash
cp gitlab_runners.yml.example gitlab_runners.yml
nano gitlab_runners.yml
cd ..
```

### 5. Установите зависимости Ansible

```bash
ansible-galaxy install -r requirements.yml
```

### 6. Запустите плейбук

```bash
ansible-playbook deploy.yml
```

### 7. Зайдите в Nexus

Необходимо принять `License Agreement` при первом входе, а также разрешить `Anonymous Access`.
Данные настройки производятся во всплывающем окне после успешного входа.

P.S. Эти параметры невозможно настроить через API, поэтому требуется ручное вмешательство

### 8. Создайте ранер в Gitlab UI

Для глобального ранера: Перейдите в `Admin` -> `CI/CD` -> `Runners` -> `Create instance runner`.
Скопируйте `runner authentication token`

### 9. Настройте блок `GitLab Runner configuration` в all.yml

```bash
cd group_vars
nano all.yml
"install_gitlab_runners: true
gitlab_runner_runners/token: скопированный ранее токен"
cd ..
```

### 10. Запустите плейбук

```bash
ansible-playbook deploy.yml
```

## 🚀 Использование

### Поведение по умолчанию:

#### Nexus

- Админ панель Nexus: `http://<NEXUS_IP>:8081`
- docker-proxy репозиторий (прокси для Docker Hub): `http://<NEXUS_IP>:5000`
- docker-hosted репозиторий (локальный репозиторий): `http://<NEXUS_IP>:5001`
- golang-proxy репозиторий (прокси для Go Modules): `http://<NEXUS_IP>:8081/repository/go-proxy/`
- Анонимный пользователь имеет права только на чтение golang-proxy репозитория

#### GitLab

- GitLab UI: `http://<GITLAB_IP>/`

#### GitLab Runner

- Раннеры автоматически регистрируются в GitLab по токену

#### Пример CI/CD пайплана находится по пути `Stable-Build/app_example`

## TODO

- [ ] Добавить поддержку SSL
- [ ] Добавить отдельного пользователя для Nexus