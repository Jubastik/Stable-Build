# Автономная CI/CD инфраструктура с GitLab, Nexus и Docker

Решение для изолированной сборки Docker-образов с кэшированием зависимостей в условиях ограниченного доступа к интернету.

## 📌 Особенности

- **Полная изоляция**: Возможна работа без доступа к Docker Hub и внешним репозиториям Go
- **Кэширование артефактов**:
  - Docker-образы через Nexus Proxy (Docker Hub mirror)
  - Golang модули через Go Proxy
- **Интеграция GitLab CI/CD**: Автоматизация тестирования и сборки

## 🏗 Архитектура

```plaintext
+----------------+        +-----------------+        +---------------+
|   GitLab CE    |        |  GitLab Runner  |        |   Nexus 3     |
| (Code Hosting) |<------>| (CI/CD Executor)|<------>| (Artifacts &  |
|                |        |      DinD       |        |  Dependencies)|
+----------------+        +-----------------+        +---------------+
```

### Компоненты
1. **GitLab CE** (Docker контейнер):
   - Хостинг репозиториев
   - Управление пайплайнами
   - Конфигурация: `http://<IP>:80`

2. **GitLab Runner** (Docker контейнер):
   - Привилегированный режим
   - Поддержка `docker:dind`

3. **Nexus 3** (Docker контейнер):
   - Репозитории:
     - `docker-hosted` (порт 5001)
     - `docker-proxy` (порт 5000)
     - `golang-proxy`

## ⚙️ Установка

### Предварительные требования
- 3 ВМ с Debian 12
- Docker и Docker Compose
- Статические IP для каждой ВМ
- Минимум 4 ГБ ОЗУ на каждой ВМ

### 1. Развертывание GitLab
Конфигурация Docker Compose: [gitlab/docker-compose.yml](gitlab/docker-compose.yml)

```bash
docker compose up -d
```
После запуска:
1. Настройте администратора через веб-интерфейс
2. Подключите ранер

### 2. Настройка GitLab Runner
Конфигурация Docker Compose: [runner/docker-compose.yml](gitlab_runner/docker-compose.yml)

```bash

Важные настройки:
```json
// /etc/docker/daemon.json
{
  "insecure-registries": ["<nexus_ip>:5000", "<nexus_ip>:5001"]
}
```

### 3. Установка Nexus
Конфигурация Docker Compose: [nexus/docker-compose.yml](nexus/docker-compose.yml)

```bash

После запуска:
1. Создайте репозитории:
   - Docker (hosted, proxy и группу для них)
   - Golang (proxy)
2. Настройте права доступа
```

## 🔄 Интеграция CI/CD

### Пример .gitlab-ci.yml
```yaml
stages:
  - test
  - build

variables:
  # Настройки
  REGISTRY_URL: "192.168.31.132:5001"
  PROXY_REGISTRY: "192.168.31.132:5000"
  DOCKER_IMAGE: "example-go-app"
  DOCKERFILE: "Dockerfile.multistage"

  # Безопасные переменные (должны быть заданы в Settings -> CI/CD -> Variables)
  CI_REGISTRY_USER: $CI_REGISTRY_USER
  CI_REGISTRY_PASSWORD: $CI_REGISTRY_PASSWORD

  # Аунтификация в registry для использования в CI/CD
  DOCKER_AUTH_CONFIG: $DOCKER_AUTH_CONFIG

  # Прокси для GO
  GOPROXY: "http://192.168.31.132/repository/golang-cache"

  DOCKER_HOST: tcp://docker:2375
  DOCKER_TLS_CERTDIR: ""



image: $PROXY_REGISTRY/docker:27.5.1
services:
  - name: $PROXY_REGISTRY/docker:27.5.1-dind
    alias: docker
    command:

      [
        "--insecure-registry", "192.168.31.132:5000",

        "--insecure-registry", "192.168.31.132:5001",

        "--registry-mirror", "http://192.168.31.132:5000"
      ]

test:
  stage: test
  image: $PROXY_REGISTRY/golang:1.19
  script:
    - go mod download
    - go test -v -race ./...

build:
  stage: build

  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $REGISTRY_URL

  script:
    - docker build --build-arg GOPROXY=$GOPROXY --pull --rm -f $DOCKERFILE -t $REGISTRY_URL/$DOCKER_IMAGE:latest .
    - docker tag $REGISTRY_URL/$DOCKER_IMAGE:latest $REGISTRY_URL/$DOCKER_IMAGE:$CI_COMMIT_SHA

    - echo "push в Docker"
    - docker push $REGISTRY_URL/$DOCKER_IMAGE:latest
    - docker push $REGISTRY_URL/$DOCKER_IMAGE:$CI_COMMIT_SHA

  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```


## 🚀 Использование

1. Запуск пайплайна:
   ```bash
   git push origin main
   ```

2. Проверка артефактов в Nexus:
   - Docker-образы: `http://<nexus_ip>:5001`
   - Кэш Go модулей: `http://<nexus_ip>/repository/golang-proxy`

## 🛠 Тонкости настройки
Для использования Docker Registry необходимо настроить **Insecure Registry** в Docker Daemon.
  ```json
  // /etc/docker/daemon.json
  {
    "insecure-registries": ["nexus_ip:5000", "nexus_ip:5001"]
  }
  ```