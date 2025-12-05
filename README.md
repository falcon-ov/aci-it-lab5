

создадим проект в GCP, зарезервируем static IP и настроим firewall;

запустим VM (Ubuntu 22.04), установим Docker;

поднимем GitLab CE в Docker (self-hosted) и настроим root;

установим и зарегистрируем GitLab Runner;

создадим Laravel-проект, Dockerfile и .gitlab-ci.yml;

запустим pipeline, проверим тесты и сборку образа;

(опционально) настроим деплой на отдельную VM или включим Container Registry.
В конце — инструкции по отладке и очистке.

Предварительно (в GCP Console или через gcloud)

В GCP Console: создать Project (или использовать существующий).

Включить биллинг для проекта.

Установить Cloud SDK на локалку, если будешь выполнять gcloud команды:

# авторизация
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

Этап A — Зарезервировать static IP и правила фаервола

Рекомендую статический внешний IP, чтобы GitLab был постоянным адресом.

# reserve static ip
gcloud compute addresses create gitlab-ip --region=us-central1
gcloud compute addresses describe gitlab-ip --region=us-central1
# смотри поле "address" — это твой STATIC_IP
`address: 34.61.112.162`
![img](/images/img_1.png)

Открыть порты для VM (HTTP, HTTPS, SSH, кастомный 8022 для GitLab SSH):

# создать правило фаервола
gcloud compute firewall-rules create allow-gitlab-ports \
  --allow tcp:80,tcp:443,tcp:22,tcp:8022 \
  --target-tags=gitlab-server \
  --description="Allow GitLab access"

![img](/images/img_2.png)

Этап B — Создать VM (Ubuntu 22.04) с Docker

Создадим VM с 4 vCPU и 16GB RAM (или больше, если есть возможность).
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
![img](/images/img_3.png)

Подключись по SSH:

gcloud compute ssh gitlab-vm-5 --zone=us-central1-a

Проверки на VM:

docker --version
docker-compose --version
ip addr show       # посмотреть локальные адреса

![img](/images/img_4.png)


Этап C — Поднять GitLab CE в Docker (быстрый и надёжный способ)

Я использую docker run, как в методичке, но с твоим STATIC_IP.

На VM:

# заменяй <STATIC_IP> на value из gcloud compute addresses describe - 34.61.112.162
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
![img](/images/img_5.png)

Проверить логи и дождаться готовности:

`sudo docker logs -f gitlab`
# дождись строки типа "GitLab now running" или "GitLab is ready"


Установить initial root password:

sudo docker exec -it gitlab cat /etc/gitlab/initial_root_password
`6hP0GbCIcjAXgQSy9qak/tPYND9qPte74F/q0JqPcsU=`
![img](/images/img_6.png)
![img](/images/img_7.png)

Открой в браузере http://<STATIC_IP> - http://34.61.112.162  и войди как root с паролем из файла, затем поменяй пароль.

Если хочешь HTTPS — позже можно привязать сертификат (letsencrypt) через Omnibus, но для задачи HTTP хватает.

Этап D — Установить и зарегистрировать GitLab Runner

На той же VM (или отдельной — лучше на другой VM для изоляции) установи runner:

# на VM
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
![img](/images/img_8.png)

В GitLab UI:

admin -> Overview / Admin Area (если root вошёл) -> CI/CD > Runners -> New instance runner

Создай runner (executor: docker), поставь флажок Run untagged jobs, скопируй Authentication Token (glrt-...).
![img](/images/img_9.png)

`glrt-DPqDDaEWWoiwrnfTjJcN5W86MQp0OjEKdToxCw.01.1216hzwuy`

Затем в VM зарегистрируй runner (пример):

sudo gitlab-runner register \
  --non-interactive \
  --url "http://34.61.112.162/" \
  --registration-token "glrt-DPqDDaEWWoiwrnfTjJcN5W86MQp0OjEKdToxCw.01.1216hzwuy" \
  --executor "docker" \
  --description "laravel-runner" \
  --tag-list "docker,php" \
  --docker-image "php:8.2-cli" \
  --run-untagged="true"
![img](/images/img_10.png)

Запусти сервис:

sudo systemctl enable gitlab-runner
sudo systemctl start gitlab-runner
sudo gitlab-runner status
![img](/images/img_11.png)


Проверка в UI: Runner должен быть активен (Admin Area → Runners).
![img](/images/img_12.png)

Этап E — Подготовка Laravel-проекта и репозитория

На твоей локалке:

::TODO BELOW

# клонируем пустой репозиторий после создания проекта в GitLab
git clone http://34.61.112.162/root/laravel-app.git
cd laravel-app

`
login: root
password: impact7723!
`

# или если возьмешь шаблон Laravel
git clone https://github.com/laravel/laravel.git ../laravel-temp
cp -r ../laravel-temp/* ./


Создай Dockerfile (в корне проекта):

# Dockerfile
FROM php:8.2-apache

RUN apt-get update && apt-get install -y \
    libpng-dev libonig-dev libxml2-dev zip unzip git \
 && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

COPY . /var/www/html
WORKDIR /var/www/html

RUN composer install --no-scripts --no-interaction --prefer-dist || true
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache
RUN chmod -R 775 /var/www/html/storage

RUN a2enmod rewrite
EXPOSE 80
CMD ["apache2-foreground"]


Создай .env.testing (пример из методички). Создай простой тест tests/Unit/ExampleTest.php:

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

Этап F — .gitlab-ci.yml (пример)

Помещаем в корень проекта. Это минимально-работающий конфиг: запуск тестов и сборка образа.

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
  image: docker:24.0.0  # runner должен поддерживать docker-in-docker, runner config может потребовать privileged
  services:
    - docker:24.0.0-dind
  variables:
    DOCKER_DRIVER: overlay2
  before_script:
    - echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin || true
  script:
    - docker build -t $CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA .
    # пример пуша в Docker Hub (альтернатива: GitLab Container Registry)
    - docker tag $CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA $DOCKERHUB_USER/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKERHUB_USER/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA
  only:
    - main


Примечания:

Для сборки образа runner должен быть настроен на docker executor и поддерживать docker:dind (часто требуется privileged = true в конфиге /etc/gitlab-runner/config.toml).

Чтобы пушить в Docker Hub — добавь переменные CI/CD в GitLab: DOCKERHUB_USER, DOCKERHUB_PASS.

Если хочешь использовать встроенный GitLab Container Registry, см. блок "Опция: Container Registry" ниже.

Этап G — Commit & push
git add .
git commit -m "Add Laravel app with CI/CD"
git push origin main


Перейди в GitLab UI → проект → CI/CD → Pipelines — пайплайн должен стартануть автоматически.

Проверяй логи job'ов: test и build.

Опция: как включить Container Registry (если нужен)

У self-hosted GitLab registry требует дополнительной конфигурации (omnibus config или запуск отдельного контейнера registry и указание registry_external_url). Коротко:

при запуске GitLab через Omnibus/docker run нужно добавить в GITLAB_OMNIBUS_CONFIG переменные для registry, например:

gitlab_rails['registry_enabled'] = true
registry_external_url 'http://<STATIC_IP>:5000'


затем запустить контейнер registry (или использовать встроенные шаги Omnibus).
Это сложнее и может требовать корректных DNS/сертификатов. Если цель — показать, что образ можно просмотреть, проще временно использовать Docker Hub или GitLab Pages.

(Если хочешь — я дам точный конфиг и docker-compose для registry.)

Этап H — Проверка и отладка

Если пайплайн в pending — проверь, что Runner активен и теги совпадают.

sudo gitlab-runner status
sudo gitlab-runner verify


Если docker:dind не запускается — в /etc/gitlab-runner/config.toml поставь privileged = true для этого runner и перезапусти gitlab-runner.

Ошибки composer/phpunit — проверь версии PHP, расширения и корректность .env.testing.

Этап I — (опционально) Автоматический деплой на другую VM

Создай вторую VM (app-vm), установи Docker и открой порты.

В build job после пуша добавь ssh-скрипт, который по успешной сборке доставляет docker-compose файл на app-vm и запускает docker-compose pull && docker-compose up -d.

В GitLab добавь CI/CD variables: DEPLOY_USER, DEPLOY_HOST, DEPLOY_SSH_KEY (private key), и используй их в job для ssh.

Пример snippet для deploy job:

deploy:
  stage: deploy
  image: appleboy/drone-ssh # можно другой образ с ssh
  script:
    - mkdir -p ~/.ssh
    - echo "$DEPLOY_SSH_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST "docker pull $DOCKERHUB_USER/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA && docker stop app || true && docker rm app || true && docker run -d --name app $DOCKERHUB_USER/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA"
  only:
    - main

Полезные команды для управления (cleanup)

Остановить и удалить GitLab:

docker stop gitlab && docker rm gitlab
docker volume rm gitlab-data


Освободить статический IP:

gcloud compute addresses delete gitlab-ip --region=us-central1


Удалить VM:

gcloud compute instances delete gitlab-vm --zone=us-central1-a

Советы по ресурсам и затратам

Выдели минимум 4GB RAM для VM; GitLab + Runner на 4GB работает, но может быть медленно.

Останови VM, когда не используешь — чтобы не платить.

В отчёте укажи точные значения: зона, машина, disk size, дата создания/удаления, итоговая стоимость за период.

Шаблон отчёта (коротко для включения в лабораторную)

Цель и краткое описание.

Конфигурация GCP (project id, zone, machine type, static IP).

Команды установки GitLab CE и Runner.

Содержимое Dockerfile, .env.testing, tests/Unit/ExampleTest.php, .gitlab-ci.yml.

Скриншоты: GitLab UI (pipeline), logs job'ов.

Проблемы и решения (если были).

Итог / как запустить вручную / cleanup.

Если хочешь, дальше могу:

сгенерировать готовый config.toml для runner с privileged = true и объяснить где поправить;

подготовить docker-compose.yml, который запускает GitLab CE + registry + postgres/redis в одном файле;

написать полный текст отчёта (на русском) с командами и скриншотами-подсказками.

Что делаем дальше — сразу генерирую нужные файлы (Dockerfile, .gitlab-ci.yml, deploy job или docker-compose) и дам точные команды для копирования на твою VM. Какой файл хочешь получить в первую очередь?