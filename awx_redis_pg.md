Вот актуальная информация о последних версиях Docker-образов для **AWX**, **Redis** и **PostgreSQL** (на **2024 год**):

---

### **1. AWX (Ansible AWX)**
- **Официальный образ**: `quay.io/ansible/awx` (Docker Hub устарел, используется Quay.io).  
- **Последняя версия**: Проверьте на [GitHub AWX Releases](https://github.com/ansible/awx/releases) (например, `23.0.0`).  
- **Тег для Docker**:  
  ```bash
  quay.io/ansible/awx:latest  # или конкретная версия, например quay.io/ansible/awx:23.0.0
  ```

---

### **2. Redis** (используется для кеширования и сообщений в AWX)
- **Официальный образ**: `redis` (на Docker Hub).  
- **Рекомендуемая версия**: `7.0` или новее.  
- **Тег для Docker**:  
  ```bash
  docker pull redis:7.0-alpine  # или redis:latest
  ```

---

### **3. PostgreSQL** (основная база данных для AWX)
- **Официальный образ**: `postgres` (на Docker Hub).  
- **Рекомендуемая версия**: AWX требует **PostgreSQL 12+**, лучше использовать `postgres:13` или `postgres:16`.  
- **Тег для Docker**:  
  ```bash
  docker pull postgres:16  # или postgres:13-alpine
  ```

---

### **Пример `docker-compose.yml` для AWX + Redis + PostgreSQL**
```yaml
version: '3'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: awx
      POSTGRES_PASSWORD: awxpass
      POSTGRES_DB: awx
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7.0-alpine
    command: redis-server --requirepass awxpass
    volumes:
      - redis_data:/data

  awx:
    image: quay.io/ansible/awx:23.0.0  # укажите актуальную версию
    depends_on:
      - postgres
      - redis
    environment:
      AWX_POSTGRES_HOST: postgres
      AWX_POSTGRES_PORT: 5432
      AWX_POSTGRES_USER: awx
      AWX_POSTGRES_PASSWORD: awxpass
      AWX_POSTGRES_DATABASE: awx
      AWX_REDIS_HOST: redis
      AWX_REDIS_PORT: 6379
      AWX_REDIS_PASSWORD: awxpass
    ports:
      - "8052:8052"
    volumes:
      - awx_data:/var/lib/awx

volumes:
  postgres_data:
  redis_data:
  awx_data:
```

---

### **Важные замечания**:
1. **AWX**:  
   - Для продакшена лучше использовать **AWX Operator** в Kubernetes.  
   - В Docker-развертывании могут быть ограничения (например, для масштабирования).  

2. **Redis**:  
   - AWX использует Redis для кеширования и Celery (очереди задач).  
   - Пароль (`requirepass`) обязателен для безопасности.  

3. **PostgreSQL**:  
   - AWX официально поддерживает только PostgreSQL (не MySQL).  
   - Версия `postgres:16` совместима, но можно использовать и `postgres:13` для проверенных конфигураций.  

---

### **Как проверить актуальные версии?**
1. **AWX**:  
   ```bash
   curl -s https://api.github.com/repos/ansible/awx/releases/latest | grep tag_name
   ```
2. **Redis**:  
   ```bash
   docker pull redis:latest && docker inspect redis:latest | grep -i version
   ```
3. **PostgreSQL**:  
   ```bash
   docker pull postgres:latest && docker inspect postgres:latest | grep -i version
   ```

Если вам нужна помощь с настройкой — спрашивайте! 🚀