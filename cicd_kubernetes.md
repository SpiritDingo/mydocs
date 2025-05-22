🔁 CI/CD и Kubernetes: как внедрить лучшие практики контейнеров

 ☸️ Пример манифеста для Kubernetes


apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: your-app
  template:
    metadata:
      labels:
        app: your-app
    spec:
      containers:
      - name: app
        image: yourrepo/app:latest
        resources:
          limits:
            memory: "512Mi"
            cpu: "1000m"
          requests:
            memory: "256Mi"
            cpu: "500m"
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          capabilities:
            drop:
              - ALL
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10


🛡 Безопасность, ограничение ресурсов, probes — всё в боевом шаблоне.

---

### 🛠 Пример CI-пайплайна (GitLab CI)


stages:
  - build
  - test
  - push

variables:
  DOCKER_BUILDKIT: 1

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build --target production -t yourrepo/app:$CI_COMMIT_SHA .
    - docker tag yourrepo/app:$CI_COMMIT_SHA yourrepo/app:latest

test:
  stage: test
  image: yourrepo/app:$CI_COMMIT_SHA
  script:
    - ./run_tests.sh

push:
  stage: push
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin
    - docker push yourrepo/app:$CI_COMMIT_SHA
    - docker push yourrepo/app:latest
  only:
    - main


⚡ Использует:
- BuildKit для кэширования и ускорения сборки  
- multi-stage Dockerfile  
- Тестирует контейнер перед пушем  
- Пушит только из main

📌 На заметку
- Используй opa-gatekeeper или Kyverno для валидации безопасности в Kubernetes  
- В CI сканируй образы через Trivy, Grype или Snyk  
- Для прода лучше использовать immutable теги, например yourrepo/app:1.4.23

🧠 Контейнер — это артефакт, который живёт дальше: в k8s, CI/CD, observability. Поэтому его здоровье — это здоровье всего DevOps-процесса.