# Настройка корпоративного npm-репозитория в Nexus с авторизацией

## Подготовка Nexus Repository Manager

1. **Создание хранилища**:
   - Войдите в Nexus с правами администратора
   - Перейдите: "Repository" → "Repositories" → "Create repository"
   - Для приватных пакетов: выберите "npm (hosted)"
   - Для проксирования npmjs.org: выберите "npm (proxy)"
   - Для группового репозитория: "npm (group)"

2. **Рекомендуемые настройки хранилища**:
   ```
   Name: npm-internal (для hosted)
   Blob store: default или создайте отдельный
   Version policy: Mixed
   Deployment policy: Allow redeploy (для CI/CD)
   Strict Content Type Validation: true
   ```

## Настройка аутентификации и авторизации

1. **Создайте пользователей**:
   - Разделите доступ на:
     - Разработчики (чтение/публикация)
     - CI-серверы (только публикация)
     - Анонимный доступ (если нужно, только чтение)

2. **Настройте роли**:
   ```
   nx-npm-read - разрешение на чтение npm-репозиториев
   nx-npm-publish - разрешение на публикацию
   nx-npm-delete - разрешение на удаление (только для админов)
   ```

3. **Настройте Realm**:
   - Активируйте "npm Bearer Token Realm" в Security → Realms

## Интеграция с npm клиентом

1. **Настройка .npmrc для разработчиков**:
   ```ini
   registry=http://nexus.your-company.com/repository/npm-group/
   //nexus.your-company.com/repository/npm-group/:_authToken=${NPM_TOKEN}
   always-auth=true
   ```

2. **Создание токена доступа**:
   - Разработчики могут получить токен через:
     ```bash
     curl -u username:password -X GET 'http://nexus.your-company.com/service/rest/v1/security/user-tokens'
     ```
   - Или через UI Nexus: "User" → "User Token"

## CI/CD интеграция

1. **Настройка переменных окружения**:
   ```bash
   export NPM_TOKEN="your-auth-token"
   ```

2. **Пример .npmrc для CI**:
   ```ini
   registry=http://nexus.your-company.com/repository/npm-internal/
   //nexus.your-company.com/repository/npm-internal/:_authToken=${NPM_CI_TOKEN}
   //nexus.your-company.com/repository/npm-group/:_authToken=${NPM_CI_TOKEN}
   always-auth=true
   ```

## Публикация пакетов

1. **Настройка package.json**:
   ```json
   {
     "publishConfig": {
       "registry": "http://nexus.your-company.com/repository/npm-internal/"
     }
   }
   ```

2. **Команда публикации**:
   ```bash
   npm publish --registry=http://nexus.your-company.com/repository/npm-internal/
   ```

## Мониторинг и обслуживание

1. **Задачи очистки**:
   - Настройте регулярную очистку старых версий пакетов
   - Конфигурируйте политики хранения в Blob Stores

2. **Резервное копирование**:
   - Регулярно делайте бэкапы blob-хранилищ и БД Nexus

## Безопасность

1. **Рекомендации**:
   - Настройте HTTPS для всех репозиториев
   - Регулярно обновляйте Nexus
   - Аудит доступа через Security → Audit
   - Настройка Webhook для уведомлений о критических действиях

## Документация для разработчиков

Создайте внутреннюю wiki-страницу с:
- Инструкцией по настройке npm клиента
- Примером публикации пакета
- Политиками именования пакетов (например, `@company-name/package-name`)
- Процедурой запроса доступа