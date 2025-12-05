# Отчет по лабораторной работе: Расширенный пайплайн в GitLab CI для Laravel

**Цель:** Получить практический опыт настройки собственного CI/CD-сервера с GitLab Community Edition и реализации конвейера для Laravel-приложения, включая тестирование, сборку Docker-образа и (опционально) деплой.

---

## 1. Развертывание GitLab CE

### 1.1 Подготовка инфраструктуры в GCP

Для развертывания GitLab CE была использована платформа Google Cloud Platform. Предварительно выполнены следующие действия:

- Создан проект в GCP Console
- Включен биллинг для проекта
- Установлен Cloud SDK на локальную машину

```bash
# Авторизация в GCP
gcloud auth login
gcloud config set project YOUR_PROJECT_ID
```

### 1.2 Резервирование статического IP-адреса

Для обеспечения постоянного доступа к GitLab зарезервирован статический внешний IP:

```bash
gcloud compute addresses create gitlab-ip --region=us-central1
gcloud compute addresses describe gitlab-ip --region=us-central1
```

Получен адрес: `34.61.112.162`

![Резервирование статического IP](/images/img_1.png)

### 1.3 Настройка правил файрвола

Открыты необходимые порты для работы GitLab:

```bash
gcloud compute firewall-rules create allow-gitlab-ports \
  --allow tcp:80,tcp:443,tcp:22,tcp:8022 \
  --target-tags=gitlab-server \
  --description="Allow GitLab access"
```

![Правила файрвола](/images/img_2.png)

### 1.4 Создание виртуальной машины

Развернута VM с Ubuntu 22.04 Server с характеристиками:
- 4 vCPU
- 16 GB RAM
- 200 GB дискового пространства

```bash
gcloud compute instances create gitlab-vm-5 \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --boot-disk-size=200GB \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --tags=gitlab-server \
  --address=gitlab-ip \
  --metadata=startup-script='#! /bin/bash
    apt-get update
    apt-get install -y docker.io docker-compose git
    usermod -aG docker $USER || true
  '
```

![Создание VM](/images/img_3.png)

Подключение к VM:

```bash
gcloud compute ssh gitlab-vm-5 --zone=us-central1-a
```

Проверка установки Docker:

```bash
docker --version
docker-compose --version
ip addr show
```

![Проверка Docker](/images/img_4.png)

### 1.5 Установка GitLab CE через Docker

Запуск GitLab CE в контейнере Docker:

```bash
sudo docker run -d \
  --hostname 34.61.112.162 \
  -p 80:80 \
  -p 443:443 \
  -p 8022:22 \
  --name gitlab \
  -e GITLAB_OMNIBUS_CONFIG="external_url='http://34.61.112.162'; gitlab_rails['gitlab_shell_ssh_port']=8022" \
  -v gitlab-data:/var/opt/gitlab \
  -v ~/gitlab-config:/etc/gitlab \
  gitlab/gitlab-ce:latest
```

![Запуск GitLab](/images/img_5.png)

Мониторинг логов до полной инициализации:

```bash
sudo docker logs -f gitlab
```

Получение начального пароля root:

```bash
sudo docker exec -it gitlab cat /etc/gitlab/initial_root_password
```

Полученный пароль: `6hP0GbCIcjAXgQSy9qak/tPYND9qPte74F/q0JqPcsU=`

![Начальный пароль](/images/img_6.png)

Вход в веб-интерфейс по адресу `http://34.61.112.162` с учетными данными root. После первого входа пароль был изменен на: `impact7723!`

![Веб-интерфейс GitLab](/images/img_7.png)

---

## 2. Настройка Runner

### 2.1 Установка GitLab Runner

На виртуальной машине установлен GitLab Runner:

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
```

![Установка Runner](/images/img_8.png)

### 2.2 Регистрация Runner

В веб-интерфейсе GitLab выполнены следующие действия:
- Переход в **Admin Area > CI/CD > Runners > New instance runner**
- Создан runner с параметрами:
  - Executor: docker
  - Description: laravel-runner
  - Tags: docker, php
  - Включена опция: Run untagged jobs

![Создание Runner](/images/img_9.png)

Получен Authentication Token: `glrt-DPqDDaEWWoiwrnfTjJcN5W86MQp0OjEKdToxCw.01.1216hzwuy`

Регистрация runner на VM:

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "http://34.61.112.162/" \
  --registration-token "glrt-DPqDDaEWWoiwrnfTjJcN5W86MQp0OjEKdToxCw.01.1216hzwuy" \
  --executor "docker" \
  --description "laravel-runner" \
  --tag-list "docker,php" \
  --docker-image "php:8.2-cli" \
  --run-untagged="true"
```

![Регистрация Runner](/images/img_10.png)

Запуск и проверка статуса:

```bash
sudo systemctl enable gitlab-runner
sudo systemctl start gitlab-runner
sudo gitlab-runner status
```

![Статус Runner](/images/img_11.png)

Проверка активности runner в веб-интерфейсе:

![Активный Runner](/images/img_12.png)

---

## 3. Создание проекта и репозитория в GitLab

### 3.1 Создание проекта

В GitLab создан новый проект: **Repository > New > Create blank project** с именем `laravel-app`.

![Создание проекта](/images/img_13.png)

### 3.2 Клонирование и настройка Laravel-проекта

Клонирование репозитория:

```bash
git clone http://34.61.112.162/root/laravel-app.git
cd laravel-app
```

Учетные данные для Git:
- Login: root
- Password: impact7723!

Копирование шаблона Laravel:

```bash
git clone https://github.com/laravel/laravel.git ../laravel-temp
cp -r ../laravel-temp/* ./
```

![Настройка проекта](/images/img_14.png)

### 3.3 Создание Dockerfile

Создан `Dockerfile` для сборки образа приложения:

```dockerfile
# Используем официальный образ PHP с Apache
FROM php:8.2-apache

# Устанавливаем зависимости
RUN apt-get update && apt-get install -y \
    libpng-dev libonig-dev libxml2-dev zip unzip git \
 && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath

# Устанавливаем Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Копируем код приложения
COPY . /var/www/html
WORKDIR /var/www/html

RUN composer install --no-scripts --no-interaction --prefer-dist || true
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache
RUN chmod -R 775 /var/www/html/storage

# Настраиваем Apache
RUN a2enmod rewrite
EXPOSE 80
CMD ["apache2-foreground"]
```

### 3.4 Конфигурация окружения для тестирования

Создан файл `.env.testing`:

```env
APP_NAME="LaravelApp"
APP_ENV=testing
APP_KEY=base64:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
APP_DEBUG=true
APP_URL=http://localhost

# Database
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=testing_db
DB_USERNAME=root
DB_PASSWORD=root

# Drivers
CACHE_DRIVER=array
SESSION_DRIVER=array
QUEUE_CONNECTION=sync

# Logging
LOG_CHANNEL=stack
LOG_LEVEL=debug

# Mail
MAIL_MAILER=array
```

### 3.5 Создание тестов

Добавлен базовый unit-тест в `tests/Unit/ExampleTest.php`:

```php
<?php
namespace Tests\Unit;
use PHPUnit\Framework\TestCase;

class ExampleTest extends TestCase
{
    public function testBasicTest()
    {
        $this->assertTrue(true);
    }
}
```

![Структура проекта](/images/img_15.png)

---

## 4. Настройка CI/CD конвейера

### 4.1 Создание .gitlab-ci.yml

Создан файл `.gitlab-ci.yml` с двумя стадиями: тестирование и сборка образа:

```yaml
stages:
  - test
  - build

services:
  - name: mysql:8.0
    alias: mysql

variables:
  MYSQL_DATABASE: laravel_test
  MYSQL_ROOT_PASSWORD: root
  DB_HOST: mysql
  DB_USERNAME: root
  DB_PASSWORD: root

cache:
  paths:
    - vendor/

test:
  stage: test
  image: php:8.2-cli
  services:
    - name: mysql:8.0
      alias: mysql
  before_script:
    - apt-get update -yqq
    - apt-get install -yqq libpng-dev libonig-dev libxml2-dev unzip git zip
    - docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath || true
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install --no-interaction --prefer-dist --no-scripts
    - cp .env.testing .env
    - php artisan key:generate
    - php artisan migrate --seed --force || true
  script:
    - vendor/bin/phpunit --stop-on-failure
  artifacts:
    when: always
    paths:
      - storage/logs/

build:
  stage: build
  image: docker:24.0.0
  services:
    - docker:24.0.0-dind
  variables:
    DOCKER_DRIVER: overlay2
  before_script:
    - echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin || true
  script:
    - docker build -t $CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA .
    - docker tag $CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA $DOCKERHUB_USER/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKERHUB_USER/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA
  only:
    - main
```

### 4.2 Настройка Docker-in-Docker

Для корректной работы stage `build` необходимо включить privileged режим в конфигурации runner:

```bash
sudo nano /etc/gitlab-runner/config.toml
```

В секции runner добавлен параметр `privileged = true`:

![Конфигурация Runner](/images/img_16.png)

### 4.3 Настройка переменных окружения

В GitLab добавлены переменные для аутентификации в Docker Hub:
- **Settings > CI/CD > Variables > Add variable**
- `DOCKERHUB_USER` - имя пользователя Docker Hub
- `DOCKERHUB_PASS` - пароль/токен Docker Hub

![Переменные CI/CD](/images/img_17.png)

---

## 5. Запуск и проверка конвейера

### 5.1 Коммит и отправка изменений

```bash
git add .
git commit -m "Add Laravel app with CI/CD"
git push origin main
```

![Отправка изменений](/images/img_18.png)

### 5.2 Мониторинг выполнения пайплайна

После push конвейер запустился автоматически. Переход в **CI/CD > Pipelines** показал активный пайплайн:

![Список пайплайнов](/images/img_19.png)

Проверка логов каждого job:

![Детали пайплайна](/images/img_20.png)

### 5.3 Результаты выполнения

Стадия **test** успешно завершена:
- Установлены все зависимости
- Выполнена миграция базы данных
- Пройдены PHPUnit-тесты

Стадия **build** успешно завершена:
- Собран Docker-образ приложения
- Образ помечен тегом с коротким SHA коммита
- Образ отправлен в Docker Hub

![Успешное выполнение](/images/img_22.png)

### 5.4 Отладка и верификация

Проверка статуса runner:

```bash
sudo gitlab-runner status
sudo gitlab-runner verify
```

![Верификация Runner](/images/img_21.png)

---

## Выводы

В ходе выполнения лабораторной работы был успешно настроен полноценный CI/CD конвейер для Laravel-приложения с использованием GitLab Community Edition. Реализованы следующие компоненты:

1. **Инфраструктура**: развернута VM в GCP с Docker, настроены сетевые правила и статический IP
2. **GitLab CE**: установлен и настроен self-hosted GitLab сервер
3. **GitLab Runner**: зарегистрирован и настроен runner с Docker executor
4. **CI/CD Pipeline**: создан конвейер с автоматическим тестированием и сборкой Docker-образа
5. **Интеграция**: настроена отправка образов в Docker Hub

Конвейер автоматически запускается при каждом push в ветку main, выполняет unit-тесты и создает готовый к деплою Docker-образ приложения. Все стадии пайплайна выполняются успешно, что подтверждает корректность настройки CI/CD процесса.