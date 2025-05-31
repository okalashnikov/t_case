# 🔒 Вопросы по DevSecOps

[🔗 Исходный материал](https://debonair-cheetah-de5.notion.site/DevSecOps-1f9907478bd08067afe2dde4c430b7b9#1f9907478bd081349d11c1c50dee028e)

---

# 🧩 Часть 1. Анализ проблем безопасности в Kubernetes и Dockerfile

## 🚢 Kubernetes

### ⚠️ Проблемы безопасности в конфигурации:

1. **🔓 Использование `privileged: true` и `runAsUser: 0`**  
   - Контейнер запускается с привилегированными правами (`privileged: true`) и от имени root (`runAsUser: 0`), что даёт ему полный доступ к хосту.  
   - **🛠️ Исправление:** Запускать контейнер от непривилегированного пользователя (`runAsUser: 1000`) и убрать `privileged: true`.

2. **🔑 Хардкод пароля (`DB_PASSWORD`) в конфигурации**  
   - Пароль указан прямо в манифесте, что небезопасно.  
   - **🛠️ Исправление:** Использовать `Secret`:
     ```yaml
     env:
     - name: DB_PASSWORD
       valueFrom:
         secretKeyRef:
           name: db-secret
           key: password
     ```

3. **🏷️ Использование `latest`-тега (`alpine-image:latest`)**  
   - Тег `latest` может привести к непредсказуемым обновлениям.  
   - **🛠️ Исправление:** Использовать конкретную версию (`alpine-image:1.2.3`).

4. **🌐 Сервис с `NodePort` открывает порт на всех узлах**  
   - Порт `30080` доступен извне без необходимости.  
   - **🛠️ Исправление:** Использовать `ClusterIP` или `NetworkPolicy`.

5. **👥 Использование дефолтного `serviceAccountName: default`**  
   - Дефолтный ServiceAccount может иметь избыточные права.  
   - **🛠️ Исправление:** Создать отдельный ServiceAccount с минимальными правами.

#### Исправленный Kubernetes Deployment (deployment.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main-portal-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: main-portal-app
  template:
    metadata:
      labels:
        app: main-portal-app
    spec:
      containers:
      - name: main-portal-container
        image: alpine-image:v1.2.3
        ports:
        - containerPort: 80
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        securityContext:
          privileged: false
          runAsUser: 1000
          runAsNonRoot: true
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
      serviceAccountName: restricted-sa
      automountServiceAccountToken: false
---
apiVersion: v1
kind: Service
metadata:
  name: main-portal-app
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
  selector:
    app: main-portal-app
```

---

## 🐳 Dockerfile

### ⚠️ Проблемы безопасности:

1. **🔄 Устаревшая версия Node.js (`node:14`)**  
   - Node.js 14 уже не поддерживается.  
   - **🛠️ Исправление:** Использовать `node:20`.

2. **📦 Добавление всех файлов через `ADD . /app`**  
   - В образ попадают ненужные файлы.  
   - **🛠️ Исправление:** Использовать `.dockerignore` и выборочное копирование.

3. **🔓 Права `777` (`chmod -R 777 /app`)**  
   - Полный доступ для всех пользователей.  
   - **🛠️ Исправление:** Использовать `755`/`644`.

4. **📥 Загрузка скрипта из интернета (`curl ... | bash`)**  
   - Риск выполнения вредоносного кода.  
   - **🛠️ Исправление:** Локальное хранение скриптов.

5. **🧩 Копирование всех `node_modules`**  
   - В продакшн попадают dev-зависимости.  
   - **🛠️ Исправление:** `pnpm install --prod`.

6. **🏗️ Отсутствие многоэтапной сборки**  
   - Лишние слои в образе.  
   - **🛠️ Исправление:** Использовать `node:20-alpine`.

7. **👑 Запуск от root**  
   - **🛠️ Исправление:** Создать отдельного пользователя.
#### Исправленный Dockerfile:
```
# Используем официальный образ с фиксированной версией
FROM node:14.21.3-alpine as build

# Безопасные ARG по умолчанию
ARG SERVICE_NAME=gate
ARG REGISTRY_URL=registry.hub.docker.com/library/

WORKDIR /app

# Копируем только необходимые файлы
COPY package.json pnpm-lock.yaml ./
COPY ./prisma ./prisma

# Устанавливаем зависимости безопасно
RUN npm install -g pnpm@7.14.0 @nestjs/cli@8.2.5 && \
    pnpm install --frozen-lockfile && \
    chown -R node:node /app  # Правильные права

# Отдельный RUN для внешних скриптов с проверкой checksum
# RUN curl -s https://github.com/somelibrary/blob/master/etc/library.sh | bash - заменено на:
ADD --chown=node:node https://verified-domain.com/trusted-script.sh /tmp/
RUN sha256sum /tmp/trusted-script.sh | grep -q "expected-checksum" && \
    bash /tmp/trusted-script.sh && \
    rm /tmp/trusted-script.sh

RUN pnpm prisma generate && \
    nest build $SERVICE_NAME

# Финальный образ
FROM node:14.21.3-alpine

WORKDIR /app
ARG SERVICE_NAME=gate

# Создаем непривилегированного пользователя
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Копируем с правильными правами
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./package.json
COPY --from=build --chown=appuser:appgroup /app/dist/apps/${SERVICE_NAME} .
COPY --from=build --chown=appuser:appgroup /app/prisma ./prisma

USER appuser  # Запускаем от непривилегированного пользователя

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/health || exit 1

ENTRYPOINT ["node", "main.js"]
```

### 🔍 Дополнительные рекомендации:
- 🕵️‍♂️ Сканировать образы (`trivy`, `docker scan`)
- 📂 Использовать `WORKDIR` с абсолютным путём
- 🏷️ Указать `EXPOSE` для документации

---

# 🧩 Часть 2. Решение практических кейсов

## 1. 🧩 Уязвимость в библиотеке без фиксов (CVE-2024-53382)
- Реализовать санитизацию входящих данных:
  * Очистка HTML-тегов перед обработкой PrismJS
  * Использование DOMPurify или аналогичных решений
- Ограничить контекст выполнения:
  * CSP (Content Security Policy) с запретом inline-скриптов
  * Установка `trustedTypes` для контроля DOM-операций
    
## Обходное решение
```
// Патчинг уязвимого метода (пример)
if (window.Prism && !Prism._patched) {
  const originalHighlight = Prism.highlight;
  Prism.highlight = function(text, grammar, language) {
    // Санкционированная очистка текста
    const sanitizedText = mySanitizer(text); 
    return originalHighlight.call(this, sanitizedText, grammar, language);
  };
  Prism._patched = true;
}
```
## Мониторинг
- Подписаться на обновления PrismJS (GitHub/GitHub Advisory)
- Настроить автоматические проверки SCA (раз в неделю)
- Рассмотреть альтернативные подсветчики синтаксиса на будущее
  
## 2. 🔑 Случайный коммит с AWS-ключом
**Действия:**
- 🚫 Немедленный отзыв ключа в AWS IAM
- 🧹 Очистка истории Git (`git filter-repo`)
- 🔄 Принудительный push изменений
- 🛡️ Настройка pre-commit хуков (`git-secrets`)

## 3. 🔒 Ограничение доступа к Secrets в Kubernetes
**Решение:**
- 🏷️ Создание отдельного Namespace
- 👮 Настройка RBAC + ServiceAccounts
- 🔗 Использование Role + RoleBinding
- 🏦 Рассмотреть внешние хранилища (Vault, KMS)

## 4. 🏷️ Проблема с тегом "latest" в Docker
**Проблемы:**
- 🔀 Невоспроизводимость сборок
- 🎲 Непредсказуемость развёртывания
- 🛡️ Потенциальные уязвимости

**Рекомендации:**
- 🏷️ Использовать конкретные версии/диджесты
- 🤖 Автоматизировать обновления (Dependabot)

---

# 🧩 Часть 3. Автоматизация рутины

## ⚙️ "Как я автоматизировал рутину и спас 20 часов в неделю"
**Реальный кейс:**  
Преобразование ручных процессов в автоматизированные скрипты с использованием:
- 🐍 Python
- 🐚 Bash
- ⏰ Cron

**Что будет раскрыто:**
- 💻 Примеры кода (доступно для не-программистов)
- 🧠 Логика решения задач
- ⚠️ "Подводные камни" и как их избежать
- 🛠️ Применение в тестировании, администрировании и аналитике
