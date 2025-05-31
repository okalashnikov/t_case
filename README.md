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
      # Запрещаем монтирование service account token (если не требуется)
      automountServiceAccountToken: false
      # Создаем отдельный service account вместо default
      serviceAccountName: main-portal-sa
      containers:
      - name: main-portal-container
        # Используем конкретную версию образа вместо latest
        image: alpine-image:3.12
        ports:
        - containerPort: 80
        # Переносим секреты в отдельный объект Secret
        envFrom:
        - secretRef:
            name: app-secrets
        securityContext:
          # Запрещаем привилегированный режим
          privileged: false
          # Запускаем от непривилегированного пользователя
          runAsUser: 1000
          runAsGroup: 1000
          # Запрещаем эскалацию привилегий
          allowPrivilegeEscalation: false
          # Только необходимые capabilities
          capabilities:
            drop:
            - ALL
          # Только для чтения корневая FS
          readOnlyRootFilesystem: true
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
      # Добавляем политику безопасности pod'а
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
---
apiVersion: v1
kind: Service
metadata:
  name: main-portal-app
spec:
  # Используем ClusterIP вместо NodePort, если не нужен доступ извне
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: main-portal-app
---
# Отдельный объект Secret для хранения чувствительных данных
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: "IXBAczV3MDByZCEhIyEk"  # Должно быть закодировано в base64
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
# Используем официальный образ с конкретной версией LTS
FROM node:20-alpine as build

# Создаем непривилегированного пользователя
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

ARG SERVICE_NAME=gate
WORKDIR /app

# Копируем только необходимые файлы для установки зависимостей
COPY package.json pnpm-lock.yaml .npmrc ./
COPY prisma ./prisma

# Устанавливаем pnpm глобально (лучше использовать локальную версию через npx)
RUN npm install -g pnpm@8 && \
    # Устанавливаем зависимости с frozen lockfile
    pnpm install --frozen-lockfile && \
    # Меняем владельца файлов
    chown -R appuser:appgroup /app

# Переключаемся на непривилегированного пользователя
USER appuser

# Копируем остальные файлы (исключая ненужные через .dockerignore)
COPY --chown=appuser:appgroup . .

# Генерируем клиент Prisma
RUN pnpm prisma generate && \
    # Собираем приложение
    pnpm nest build $SERVICE_NAME

# Финальный образ
FROM node:20-alpine

WORKDIR /app
ARG SERVICE_NAME=gate

# Создаем пользователя в финальном образе
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Копируем только необходимые файлы из стадии build
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./
COPY --from=build --chown=appuser:appgroup /app/dist/apps/${SERVICE_NAME} .
COPY --from=build --chown=appuser:appgroup /app/prisma ./prisma

# Переключаемся на непривилегированного пользователя
USER appuser

# Запускаем приложение
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
