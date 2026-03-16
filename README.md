# Habit Tracker API

Backend-сервис для трекинга полезных привычек на Django REST Framework.
Проект реализован как API для SPA-клиента с JWT-аутентификацией, правами доступа, бизнес-валидацией и Telegram-напоминаниями через Celery.

## О проекте

Сервис помогает пользователю:

- создавать полезные и приятные привычки;
- связывать полезную привычку с приятной (как награда);
- задавать периодичность и длительность выполнения;
- публиковать привычки в общий список;
- получать напоминания в Telegram.

## Основные возможности

- JWT-аутентификация по email (`register`, `token`, `token/refresh`);
- CRUD только для привычек владельца;
- публичный read-only список привычек (требует авторизацию);
- пагинация по 5 элементов;
- валидаторы;
- CORS для подключения фронтенда;
- Swagger-документация API;
- отложенные Telegram-напоминания через Celery + Redis.

---

## Технологии

- Python 3.13+
- Django 6
- Django REST Framework
- SimpleJWT
- Celery
- Redis
- drf-yasg (Swagger)
---

## Быстрый старт
### 1) Установка зависимостей

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```
### 2) Настройка окружения

Создайте `.env` на основе `env.example`.

### 3) Миграции

```bash
python manage.py migrate
```

### 4) Запуск API

```bash
python manage.py runserver
```

API будет доступен по адресу:
```
http://localhost:8000
```
### 5) Запуск Redis (локально)

Установка на Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install -y redis-server
```

Запуск:

```bash
sudo systemctl enable --now redis-server
```

Проверка:

```bash
redis-cli ping
```

### 6) Запуск Celery

В двух отдельных окнах терминала:

```bash
celery -A config worker -l info
```
```
celery -A config beat -l info
```

---

## Документация API

```
http://localhost:8000/swagger/
```

## Аутентификация

Для защищенных endpoint используйте заголовок:

`Authorization: Bearer <access_token>`

---

## API: (Postman)
### Узнайте свой id Telegram:
Открой в Telegram бота @userinfobot (или @getmyid_bot).
Нажми Start.
Скопируйте поле 
Id / Chat ID и укажи его при регистрации.

### 1) Регистрация

POST
```
http://localhost:8000/api/users/register/
```
```json
{
  "email": "user@example.com",
  "password": "StrongPass123",
  "telegram_chat_id": "12345678"
}
```

Примечания:

- `email` обязателен и уникален;
- `telegram_chat_id` можно передать сразу или установить позже отдельным запросом.
- Регистрация без `telegram_chat_id` допустима: аккаунт создается, но Telegram-напоминания начнут работать только после сохранения `chat_id`.

### 2) Получение JWT

POST 
```
http://localhost:8000/api/users/token/
```
```json
{
  "email": "user@example.com",
  "password": "StrongPass123"
}
```

Ответ содержит `access` и `refresh`.

### 3) Обновление telegram_chat_id после регистрации

PATCH 
```
http://localhost:8000/api/users/telegram-chat-id/
```
```json
{
  "telegram_chat_id": "12345678"
}
```

Требуется JWT.

### 4) Создание привычки

POST 

```
http://localhost:8000/api/habits/
```
Пример приятной привычки:

```json
{
  "place": "Дом",
  "time": "21:35:00",
  "action": "Принять ванну",
  "is_pleasant": true,
  "periodicity": 1,
  "execution_time": 90,
  "is_public": false
}
```

Пример полезной привычки с наградой:

```json
{
  "place": "Дом",
  "time": "21:30:00",
  "action": "Читать 10 страниц",
  "is_pleasant": false,
  "periodicity": 1,
  "reward": "Чашка чая",
  "execution_time": 120,
  "is_public": false
}
```

### 5) Список своих привычек

GET

```
http://localhost:8000/api/habits/
```
Пагинация: 5 элементов на страницу.

### 6) Получение/изменение/удаление привычки

- GET ```http://localhost:8000/api/habits/<id>/```
- PATCH ```http://localhost:8000/api/habits/<id>/```
- DELETE ```http://localhost:8000/api/habits/<id>/```

### 7) Список публичных привычек

GET 
```
http://localhost:8000/api/habits/public/
```
Требуется JWT.
---

## Бизнес-правила (валидаторы)

1. Нельзя одновременно указывать `related_habit` и `reward`.
2. `execution_time` не может превышать 120 секунд.
3. В `related_habit` можно указывать только приятную привычку (`is_pleasant=true`).
4. У приятной привычки не может быть `reward` или `related_habit`.
5. Полезная привычка обязана иметь `reward` или `related_habit`.
6. `periodicity` должна быть в диапазоне от 1 до 7 дней.



Проверка:

1. Убедитесь, что запущены `worker` и `beat`.
2. Создайте привычку с актуальным временем и нужной периодичностью.
3. Дождитесь времени выполнения — уведомление придет в Telegram.



## Проверки

### Тесты

```bash
python manage.py test
```

### Покрытие

```bash
coverage run manage.py test
coverage report -m
```


Миграции исключены в [.flake8](.flake8).

### Swagger-документация

Откройте в браузере:
```url
http://localhost:8000/swagger/
```


### Проверка публичного списка с авторизацией

```bash
curl -H "Authorization: Bearer <access_token>" http://localhost:8000/api/habits/public/
```

### Проверка пагинации
GET Authorization: Bearer <access_token>

```bash
http://localhost:8000/api/habits/?page=1
```

### Проверка Telegram

Проверка доступности бота и тестовое сообщение:

```bash
python manage.py telegram_check --chat-id <chat_id> --text "Привет, Я 🤖 Habbit_Bot из твоего тестового запроса."
```

Если передать только `--chat-id`, сообщение будет отправлено со стандартным текстом.


# Docker
Установка на Debian:
```angular2html
sudo apt update

sudo apt install ca-certificates curl gnupg
```
```angular2html
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg


echo \

  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \

  bookworm stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null



```
Установка прав доступа для Docker в Linux, чтобы использовать команды без sudo.
Добавьте пользователя в группу docker, это позволит управлять контейнерами, 
образами и томами без повышения привилегий до root. 


```angular2html
sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Создайте группу docker (если она не создана):
```angular2html
sudo groupadd docker
```
Добавьте вашего пользователя в группу:

```angular2html
sudo usermod -aG docker $USER
```
Примените изменения:
```angular2html
newgrp docker
```
Проверка без sudo:
```angular2html
docker run hello-world
```
Добавление пользователя в группу docker эквивалентно предоставлению прав root,
так как позволяет контейнерам получить доступ к файловой системе хоста.

## Клонирование и развертывание проекта на локальном ПК
Создайте и перейдите в папку:
```angular2html
sudo mkdir /var/www/
cd /var/www/
```
Дайте права на чтение и запись
```
sudo chown -R $USER:$USER /var/www/
sudo chmod -R 755 /var/www/
```
В терминале введите команду для клонирования проекта находясь в папке /var/www
```angular2html
git clone https://github.com/BatiaForWorld/Course_work_last.git
```
Перейдите внутрь папки скаченного проекта:
```angular2html
cd /var/www/Course_work_last
```

## Docker Compose

### Быстрый запуск

1) Скопируйте шаблон окружения и заполните значения находясь в дериктории проекта:

```bash
cp env.example .env
nano .env
```
Сохраните файл Ctrl+O Enterи выйдите Ctrl+X

2) Запуск всех сервисов одной командой:

```bash
docker compose up --build -d
```
Проверьте в соседнем терминале:
Проверяем Time Zone:
```
docker compose exec web env | grep TIME_ZONE
```
Проверить, какие значения берёт compose для PostgreSQL:

```
docker compose config | grep -A3 POSTGRES_
```
Проверить, какие значения берёт compose для Redis:

```angular2html
docker compose config | grep -A3 REDIS_
```
Миграции применяются автоматически при старте сервиса `web`.

Если нужно пересобрать заново контейнеры, выполните команду:
```angular2html
docker compose down -v
```
И соберите заново:
```angular2html
docker compose up --build -d
```
Если нужно удалить контейнеры и созданную папку с приложение, выполните команду
```angular2html
cd /var/www/Course_work_last
docker compose down
cd /var/www/
sudo rm -r Course_work_last
cd
```
### Проверка работоспособности на локальном ПК

- Django API: открыть 
```
http://localhost/swagger/
 ```
- PostgreSQL: 
```
docker compose exec db pg_isready -U <USER> -d <NAME>
```
- Redis: 
```
docker compose exec redis redis-cli ping
```
- Celery worker:
```
docker compose logs -f celery
```
- Celery beat:
```
docker compose logs -f celery-beat
```
# CI/CD и GitHub Actions

Создайте свой VPS на сервере. 

Это может быть ваш выделенный компьютер для функций сервера или арендованный VPS у хостинг провайдера.

Для своего VPS настройте сеть для доступа ваших контейнеров к глобальной сети интернет, 
открыв порт на роутере 80/tcp
и 22/tcp для удаленного соединения по ssh. 
И внесите настройки ssh после обмена ключами 
с вашим VPS. Advanced --> NAT Forwarding --> Virtual Server

Создайте SSH ключ 
```angular2html
ssh-keygen -t ed25519 -C "email"
```
Добавьте его на ваш VPS:
```angular2html
ssh-copy-id -p 22 user@192.12ваш_ip
```
Напиши YES и введите пароль пользователя под которым вы заходите

Далее войдите в ваш VPS:
```angular2html
ssh -p 22 user@192.12ваш_ip
```

!!!Смените порт ssh по умолчанию.
Откройте файл:
```angular2html
sudo nano /etc/ssh/sshd_config
```
Найдите и измените следующие параметры (уберите #, если строка закомментирована):
- Port 22  `смените на любой выбрав в диапозоне 50000–65000`
- PasswordAuthentication no — запрещает вход по обычному паролю.
- PubkeyAuthentication yes — разрешает вход по ключам.
- PermitRootLogin prohibit-password — (рекомендуется) разрешает root-вход только по ключу.

Сохраните файл `` Ctrl+O`` `` Enter ``и выйдите ``Ctrl+X``
Чтобы настройки вступили в силу перезапустите ssh сервер:
```
sudo systemctl restart ssh
```


Если вы используете LXC контейнеры, то необходимо создать мост, что бы контейнеры могли получали адрес от 
роутера, предварительно зафиксировав MAC адрес LXC контейнера и зарезервировать его в настройках роутера 
**Advanced --> Network --> Lan Settings --> Address Reservation**,
что бы при перезагрузке сервера ваш контейнер не потерял свой ip адрес в локальной сети. 

Установите UFW(Uncomplicated Firewall) — это простой инструмент командной строки для управления 
брандмауэром (firewall) в Linux  
```angular2html
sudo apt update
sudo apt install ufw
```
Откройте порты для доступа сетевого трафика:

Добавьте правила:

Для Nginx

```angular2html
sudo ufw allow 80/tcp
```
Для SSH соединения:
```angular2html
sudo ufw allow 22
```
Если нужно закрыть порт нйдите его порядковый номер командой
```angular2html
sudo ufw status numbered
```
И удалите указав порядковый номер из списка открытых портов
```angular2html
sudo ufw delete 1
```
Не забывайте, что на арендованных VPS так же есть страница настройки правил Firewall.



### Создайте папку для проекта в вашем VPS сервере

```angular2html
sudo mkdir /var/www/habits
```
Заполните файл .env по шаблону из env.example
```angular2html
sudo nano /var/www/habits/.env
```
Сохраните файл `` Ctrl+O`` `` Enter ``и выйдите ``Ctrl+X``
Посмотрите файл .env, что бы убедиться что он создан
```angular2html
cat /var/www/habits/.env
```
Задайте права доступа для группы Docker

Дайте права на чтение и записи в папку проекта с контенерами Docker
```
sudo chown -R $USER:$USER /var/www/habits
sudo chmod -R 755 /var/www/habits
```
### Добавьте необходимые секреты для GitHub workflows:

DEPLOY_DIR - папка которая содержит проект
```angular2html
/var/www/habits
```

DOCKER_HUB_ACCESS_TOKEN - токен с Docker Hub

DOCKER_HUB_USERNAME - логин с Docker Hub

SERVER_IP - ваш публичный IP
	
SSH_KEY - ssh ключ 

Откройте и скопируйте с дефисами c вашего пк, с которого вы обменивались ключами с VPS
```angular2html
cat ~/.ssh/id_ed25519
```
SSH_PORT - порт вашего ssh

SSH_USER - имя пользователя вашего VPS

Выполните push из ветки и автоматически запуститься  action на GitHub.

Дождитесь выполнения workflows. 

### lint. . .  -->

### test. . .  -->

### build. . .  -->

### deploy. . .  !

По завершению успешного deploy приложение будет доступно по адресу

```angular2html
http://habits.help/swagger/
```

Для работы с сервисом воспользуйтесь раннее описанной инструкцией к "Django REST Framework".

## Проверки работы Docker на VPS

Перейдите в папку с проектом в терминале вашего VPS
```angular2html
cd /var/www/habits
```

Проверьте работу контейнеров

```angular2html
docker ps
```
Проверьте расход ресурсов вашйей VPS

```angular2html
docker stats
```

### Найдите нужный вам контейнер

Посмотреть контейнер по имени:
```angular2html
docker compose ps
```
Посмотреть контейнер по id:
```angular2html
docker ps
```
Перезапуск выбранного контейнера 
```angular2html
docker restart <id_контейнера или имя_контейнера>
```
### Список команд перезапуска отдельных контейнеров:
Redis
```angular2html
docker compose restart redis
```
Nginx
```angular2html
docker compose restart nginx
```
PostgreSQL
```angular2html
docker compose restart postgres
```
Celery
```angular2html
docker compose restart celery
```
Celery-beat
```angular2html
docker compose restart celery-beat
```
Gunicorn
```angular2html
docker compose restart web
```
Проверка файла конфигурации на наличие ошибок
```angular2html
docker compose exec nginx nginx -t
```
Проверка логов, которые можно настроить для fail2ban, а так же выявлять запросы к ввашему серверу на VPS
```angular2html
docker compose logs nginx --tail 20
```
Перезапуск Nginx при изменениях **config** файла без полной остановки и перезапуска контейнера.
```angular2html
docker compose exec nginx nginx -s reload
```
Проверка логов сервисов:

Redis
```angular2html
docker compose logs redis --tail 20
```
Celery:
```angular2html
docker compose logs celery --tail 20
```
Celery-beat:
```angular2html
docker compose logs celery-beat --tail 20
```
Gunicorn:
```angular2html
docker compose logs web --tail 20
```
PostgreSQL:
```angular2html
docker compose logs db --tail 20
```
Проверка, принимает ли база подключения:
```angular2html
docker compose exec db pg_isready -U <USER_NAME>
```
Redis (Проверка отклика)
```angular2html
docker compose exec redis redis-cli ping
```
Проверка, видит ли Celery воркер очередь и готов ли он к работе:

Посмотреть активные воркеры
```angular2html
docker compose exec celery celery -A config inspect active
```

Логи всего Docker compose
```angular2html
docker compose logs -f
```
Если необходимо перезапустить контейнеры
```angular2html
docker compose restart
```
Если необходимо, пересоберите контейнеры
```angular2html
docker compose down
docker compose up -d --build
```
Вышеуказаные команды выполнять находясь в папке
```angular2html
cd /var/www/habits
```
### Удаление проекта с VPS
```angular2html
cd /var/www/habits
docker compose down
cd /var/www/
sudo rm -r habits
cd
```



Автор: Казанцев Андрей
