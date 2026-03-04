# Техническая настройка — Админка Google Ads

Система автоматизации управления refresh tokens для Google Ads API с поддержкой автоматического обновления токенов на удалённых серверах.

## Возможности

- Автоматическое обновление refresh tokens через OAuth2
- Ежедневное обновление токенов по расписанию  
- Автоматическое обновление токенов в PHP скриптах на удалённых серверах
- Веб-интерфейс для мониторинга

## Быстрый запуск

### 1. Клонирование и настройка

```bash
# Переходим в директорию проекта
cd /opt/apprix_dev

# Копируем файл переменных окружения
cp env_example .env

# Редактируем переменные окружения
nano .env
```

### 2. Настройка переменных окружения

Отредактируйте `.env` файл:

```env
# Django Settings
DEBUG=False
SECRET_KEY=your-very-secret-key-here
ALLOWED_HOSTS=localhost,127.0.0.1,your-domain.com

# Database
DATABASE_URL=postgresql://apprix_user:apprix_password@db:5432/apprix_dev

# Celery
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0

# Token Manager
TOKEN_API_KEY=your-secure-api-key-here
TOKEN_ENCRYPTION_KEY=your-32-byte-base64-encoded-key

# Sentry (опционально)
SENTRY_DSN=https://your-sentry-dsn-here
```

### 3. Генерация ключа шифрования

```bash
# Генерируем ключ шифрования для токенов
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

### 4. Запуск с Docker Compose

```bash
# Сборка и запуск всех сервисов
docker-compose up -d

# Просмотр логов
docker-compose logs -f

# Проверка статуса
docker-compose ps
```

### 5. Первоначальная настройка

```bash
# Создание суперпользователя (если не создался автоматически)
docker-compose exec web python apprix_dev/manage.py createsuperuser

# Применение миграций (если нужно)
docker-compose exec web python apprix_dev/manage.py migrate
```

## Настройка Google Ads клиентов

### 1. Вход в админ панель

Перейдите на `http://localhost:8000/admin/` и войдите с правами администратора.

### 2. Добавление Google Ads клиента

1. Перейдите в `Google Ads клиенты` → `Добавить`
2. Заполните данные:
   - **Название**: Произвольное имя клиента
   - **Customer ID**: ID аккаунта Google Ads
   - **Client ID**: OAuth2 Client ID из Google Cloud Console
   - **Client Secret**: OAuth2 Client Secret
   - **Developer Token**: Developer Token от Google Ads API
   - **Login Customer ID**: ID аккаунта с правами доступа

### 3. Настройка серверов скриптов

В поле `Script Servers` добавьте JSON конфигурацию серверов:

```json
[
  {
    "host": "your-server.com",
    "username": "your-username",
    "auth_method": "key",
    "private_key_path": "/path/to/private/key",
    "script_paths": [
      "/var/www/html/1securityw2a.php",
      "/var/www/html/abranusw2a.php"
    ]
  },
  {
    "host": "another-server.com",
    "username": "user",
    "auth_method": "password",
    "password": "your-password",
    "script_paths": [
      "/home/user/scripts/magicsecw2a.php"
    ]
  }
]
```

### 4. Получение токенов

1. В админ панели или на дашборде нажмите кнопку авторизации для клиента
2. Пройдите процесс OAuth2 авторизации в Google
3. Токены будут автоматически сохранены и отправлены на указанные серверы

## Веб-интерфейсы и API

### Веб-интерфейсы

- **Дашборд**: `http://localhost:8000/tokens/`
- **Админ панель**: `http://localhost:8000/admin/`
- **Flower (мониторинг Celery)**: `http://localhost:5555/`

### API Endpoints

```bash
# Получение токена по customer_id
curl -H "Authorization: Bearer your-api-key" \
     http://localhost:8000/tokens/api/token/1234567890/

# Принудительное обновление всех токенов
curl -X POST -H "Authorization: Bearer your-api-key" \
     http://localhost:8000/tokens/api/refresh/

# Обновление конкретного клиента
curl -X POST -H "Authorization: Bearer your-api-key" \
     -H "Content-Type: application/json" \
     -d '{"client_id": 1}' \
     http://localhost:8000/tokens/api/refresh/
```

### Расписание задач Celery

- **Обновление токенов**: Каждый день в 2:00
- **Очистка логов**: Каждое воскресенье в 3:00

### Настройка расписания

Редактируйте в `settings.py`:

```python
CELERY_BEAT_SCHEDULE = {
    'refresh-all-tokens-daily': {
        'task': 'token_manager.tasks.refresh_all_tokens',
        'schedule': crontab(hour=2, minute=0),
    },
}
```

## Структура проекта

```
/opt/apprix_dev/
├── apprix_dev/                 # Django проект
│   ├── apprix_dev/            # Настройки проекта
│   └── token_manager/         # Приложение управления токенами
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── nginx.conf
└── README.md
```

## Команды Docker

```bash
# Сборка и запуск
docker-compose up -d

# Остановка
docker-compose stop

# Полная остановка
docker-compose down

# Логи
docker-compose logs -f [service_name]

# Обновление
docker-compose up -d --build

# Резервное копирование БД
docker-compose exec db pg_dump -U apprix_user apprix_dev > backup.sql

# Восстановление БД
docker-compose exec -T db psql -U apprix_user apprix_dev < backup.sql
```

## Продакшен

### С Nginx

```bash
docker-compose --profile production up -d
```

### Настройка SSL

1. Поместите SSL сертификаты в папку `ssl/`
2. Обновите `nginx.conf` для поддержки HTTPS
3. Перезапустите nginx контейнер

### Переменные окружения для продакшена

```env
DEBUG=False
ALLOWED_HOSTS=your-domain.com,www.your-domain.com
SECRET_KEY=very-long-random-secret-key
TOKEN_ENCRYPTION_KEY=base64-encoded-32-byte-key
SENTRY_DSN=your-sentry-dsn-for-error-tracking
```

## Troubleshooting

### Проблемы с БД

```bash
docker-compose ps
docker-compose logs db
docker-compose exec db psql -U apprix_user apprix_dev
```

### Проблемы с Celery

```bash
docker-compose logs celery
# Flower: http://localhost:5555
```

### Проблемы с SSH

1. Проверьте доступность: `ping your-server.com`
2. Проверьте SSH: `ssh username@your-server.com`
3. Для ключей: права 600 на приватный ключ
