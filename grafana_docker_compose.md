Если вы используете современную версию Docker Compose (v2.x), то формат файла docker-compose.yml может использовать синтаксис версии 3 или выше. Ниже приведён актуальный файл, совместимый с последними версиями Docker Compose, с использованием версии 3.8 (рекомендуется) и последнего образа Grafana.

```yaml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      # При необходимости измените или добавьте другие переменные
    volumes:
      - grafana-storage:/var/lib/grafana
    # При необходимости можно задать сеть
    # networks:
    #   - grafana-net

volumes:
  grafana-storage:
    name: grafana-data

# networks:
#   grafana-net:
#     driver: bridge
```

Запуск

1. Сохраните файл как docker-compose.yml.
2. Выполните:
   ```bash
   docker compose up -d
   ```
   (для Docker Compose v2 команда без дефиса)
3. Откройте http://localhost:3000, войдите с admin/admin.

Пояснения

· Версия 3.8 — актуальная и поддерживает все современные возможности.
· Используется том с явным именем grafana-data для сохранения данных.
· Параметр restart: unless-stopped обеспечивает автоматический перезапуск контейнера после перезагрузки хоста (если он не был остановлен вручную).

Если вы хотите зафиксировать конкретную версию Grafana вместо latest, укажите её, например grafana/grafana:10.4.0.