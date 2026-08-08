Домашнее задание №7 — Helm + GitOps: ArgoCD, SOPS-секреты, Security Context

> Эта домашка тяжёлая — по объёму как две недели. Ты переводишь приложение с «сырых» манифестов на **Helm-чарты**, шифруешь секреты через **SOPS + age** и отдаёшь деплой **ArgoCD** (GitOps). Часть тем (library chart, DevSecOps-сканы, вторая нода) вынесена в [self-study](https://it-incubator.io/react/ru/devops-for-devs/03-gitops-extras-self-study.mdx) и **не** входит в обязательную проверку — сначала закрой ядро.

## Как сдать домашку

‼️ Только для студентов Стандартного (Синхронного) тарифного плана

Отправьте своё решение через
[гугл-форму](https://docs.google.com/forms/d/e/1FAIpQLSf89bZDGX3vXAOI5oA1ZfNszadEc6_yiCLXKckuB4ggccymHA/viewform)

Домашку запушьте в свой репозиторий на GitHub:

- Репозиторий с кодом (`devops-hometask-01`) должен быть публичным, задание — в ветке `week7`.
- GitOps-репозиторий (`devops-gitops`) — **публичный** (секреты в нём зашифрованы SOPS, поэтому открытый git безопасен); ссылку сдавать не нужно — агент увидит его через ArgoCD.
- Дедлайн — **2 сентября 2026, среда, 23:59 МСК**.

## Что было в hw6 и что меняется

В hw6 приложение переехало в **k3s**: сырые манифесты `k8s/*.yaml`, деплой через `envsubst | kubectl apply`, секреты — вручную `kubectl apply -f secret.yaml`. Работает, но:

- каждый новый namespace/кластер = ручная правка домена, образа, кредов в YAML;
- секреты нигде не версионируются — непонятно, что в каком состоянии;
- CI держит kubeconfig и сам ходит в кластер (широкие права).

В hw7 ты закрываешь эти три проблемы:

> ⚠️ **Смена платформы деплоя.** hw7 удаляет путь hw6 (`k8s/*.yaml` + `deploy-k8s.yml` + `envsubst`). Роль «источника правды» для кластера переходит к git-репозиторию, а применять изменения будет ArgoCD. Легаси-проверки hw6 (сырые манифесты, старый deploy-workflow) больше не прогоняются — их заменяют Helm/ArgoCD-эквиваленты.

## Новое: два репозитория

До сих пор всё жило в одном репозитории. GitOps разводит **код** и **конфигурацию** по разным репозиториям:

ArgoCD следит за `devops-gitops`, а CI из `devops-hometask-01` лишь обновляет там один тег образа. Git — единственный источник правды: кластер всегда приведён к тому, что лежит в gitops-репозитории.

## Репозиторий (код): патч

Заготовка hw7 приезжает **патчем** поверх твоей готовой `week6`. Патч здесь тонкий — вся конфигурация теперь в gitops-репозитории, а не в code-репо.

```text
git checkout -b week7 week6
curl -LO https://raw.githubusercontent.com/it-incubator/devops-hometask-01/hw7-handoff/hw7.patch
git apply hw7.patch
git add -A
git commit -m "scaffold hw7"
rm hw7.patch
```

Что делает патч:

- **Удаляет `k8s/`** — сырые манифесты hw6 больше не нужны, их заменяют Helm-чарты в gitops-репо.
- **Добавляет `.gitleaks.toml`** — конфиг для DevSecOps-сканов (используется в self-study).

> Старый workflow `.github/workflows/deploy-k8s.yml` ты создавал сам в hw6 — патч его не трогает. Его надо **удалить руками** при создании нового пайплайна (Шаг 10): `git rm .github/workflows/deploy-k8s.yml`.

> **Файлы из hw1–hw6 патч не трогает.** `back/`, `front/`, Dockerfile-ы, `Makefile`, `front/e2e/`, `teacher_authorized_key.pub` — всё как в `week6`. Менять их не нужно.

## Pre-requisites

- **hw6 сдана и работает**: домен, HTTPS, k3s, `make e2e-deployed` зелёный. hw7 строится поверх рабочего кластера.
- **Апгрейд VPS до ≥ 4 ГБ RAM (желательно ≥ 2 vCPU)** (см. Шаг 0). ArgoCD прожорлив — на 1.8 ГБ hw6 он не встанет; на 2 CPU CPU-реквесты впритык, поэтому приложение крутим в 1 реплике.
- **Локальные инструменты**: `helm`, `age`, `sops`, плагин `helm-secrets`, CLI `argocd`. Установка — по ходу шагов.
- **Docker Hub** аккаунт с образами `todo-back`, `todo-front` (как в hw6).
- **GitHub** аккаунт (для template-репо и токенов).

## Что нужно сделать

### Шаг 0. Апгрейд VPS до 4 ГБ

ArgoCD (`repoServer` + redis + контроллеры) не поместится рядом с k3s + ingress + cert-manager + приложением на 1.8 ГБ. В панели провайдера подними тариф текущего VPS до **≥ 4 ГБ RAM**. Кластер остаётся **одноузловым** (подключение второй ноды — в self-study).

```text
# на VPS — убедись, что памяти стало больше
free -h
#               total        used        free
# Mem:          3.8Gi       1.1Gi       2.1Gi
```

### Шаг 1. Применить патч, ветка `week7`

Команды выше (`git checkout -b week7 week6` + `git apply`). После этого `k8s/` исчез, появился `.gitleaks.toml`.

### Шаг 2. Создать gitops-репозиторий из шаблона

1. Открой template-репозиторий курса **[it-incubator/devops-gitops-starter](https://github.com/it-incubator/devops-gitops-starter)** → кнопка **«Use this template» → Create a new repository**.
2. Имя: `devops-gitops`, видимость: **Public**. Секреты зашифрованы SOPS, поэтому публичный git безопасен, а ArgoCD читает его по HTTPS без ключей. **Приватный `age-key.txt` при этом коммитить нельзя ни в коем случае** — на публичном репо утечка мгновенна (см. Шаг 3, и gitleaks-CE в репо это ловит).
3. Склонируй его локально и заполни плейсхолдеры:

> 💡 Заведи DNS-запись `argocd.<домен>` (A-запись на IP VPS) — на этот сабдомен встанет UI ArgoCD.

### Шаг 3. Секреты: SOPS + age

**age** — асимметричное шифрование: публичный ключ шифрует, приватный расшифровывает. **SOPS** шифрует только *значения* в YAML, оставляя ключи читаемыми. Так зашифрованный `secrets.yaml` можно спокойно коммитить в git.

```text
# Установка (macOS)
brew install age sops
helm plugin install https://github.com/jkroepke/helm-secrets
 
# Генерация ключа
age-keygen -o age-key.txt
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
 
# SOPS ищет приватный ключ через переменную окружения
echo 'export SOPS_AGE_KEY_FILE=~/age-key.txt' >> ~/.zshrc && source ~/.zshrc
```

> ❌ **`age-key.txt` — приватный ключ. Никогда не коммить.** Он уже в `.gitignore` стартера.

Вставь **публичный** ключ в `.sops.yaml` (поле `age:`). Затем в обоих чартах создай `secrets.yaml` из примера, подставь значения и зашифруй:

```text
cp helm/postgres/secrets.example.yaml helm/postgres/secrets.yaml
cp helm/myapp/secrets.example.yaml    helm/myapp/secrets.yaml
# впиши реальный пароль БД (ОДИН и тот же в обоих файлах!)
 
sops -e -i helm/postgres/secrets.yaml
sops -e -i helm/myapp/secrets.yaml
```

> ⚠️ **Креды БД должны совпадать** в `postgres/secrets.yaml` и `myapp/secrets.yaml`: postgres инициализируется ими, backend ими же подключается.

Проверь, что значения нечитаемы, и закоммить зашифрованные файлы:

```text
grep -A1 DB_PASSWORD helm/myapp/secrets.yaml
# DB_PASSWORD: ENC[AES256_GCM,data:...]   ← зашифровано, коммитить можно
 
git add .sops.yaml helm/*/secrets.yaml helm/myapp/values.yaml application.yaml argocd-values.yaml
git commit -m "feat: charts + encrypted secrets"
git push
```

### Шаг 4. Миграция: убрать raw-ресурсы hw6

Твоё приложение hw6 сейчас развёрнуто **сырыми манифестами** (`kubectl apply`). Helm и ArgoCD не могут «усыновить» чужие ресурсы: `helm install postgres` упадёт с

```text
Error: Service "postgres" ... exists and cannot be imported into the current release:
missing key "app.kubernetes.io/managed-by": must be set to "Helm"
```

Поэтому старые raw-ресурсы надо **удалить** — Helm (postgres) и ArgoCD (myapp) создадут их заново, уже под своим управлением. **Данные БД не потеряются**: PVC `data-postgres-0` при удалении StatefulSet сохраняется, а новый postgres переиспользует его.

```text
# Удаляем raw hw6-ресурсы. PVC и dockerhub-cred НЕ трогаем.
kubectl delete deploy backend frontend --ignore-not-found
kubectl delete svc backend frontend postgres --ignore-not-found
kubectl delete statefulset postgres --ignore-not-found
kubectl delete ingress app-ingress --ignore-not-found
kubectl delete cm backend-config backend-appconfig --ignore-not-found
kubectl delete secret backend-secret --ignore-not-found
 
# Проверь, что PVC с данными на месте:
kubectl get pvc data-postgres-0
# data-postgres-0   Bound   ...   1Gi
```

> ⚠️ **Не удаляй** `pvc/data-postgres-0` (данные БД) и `secret/dockerhub-cred` (доступ к образам). На время миграции приложение будет недоступно — это нормально, Helm/ArgoCD поднимут его заново в следующих шагах.

### Шаг 5. Postgres вручную + доступ к образам

Postgres — **stateful**, у него свой жизненный цикл; под ArgoCD с `selfHeal`/`prune` держать БД опасно. Поэтому ставим его **вручную один раз**. Сначала — доступ к приватным образам Docker Hub (если `dockerhub-cred` уже есть с hw6 — пропусти):

```text
# Secret с кредами Docker Hub (namespace default) — как в hw6
kubectl create secret docker-registry dockerhub-cred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=ВАШ_ЛОГИН \
  --docker-password=dckr_pat_ВАШ_ТОКЕН
```

```text
# Postgres через helm-secrets (secrets:// расшифрует SOPS на лету)
helm install postgres ./helm/postgres \
  -f helm/postgres/values.yaml \
  -f secrets://helm/postgres/secrets.yaml
 
kubectl get pods -w
# postgres-0   1/1   Running   ← переиспользует старый PVC, данные на месте
```

> 💡 Новый `postgres-0` подхватывает существующий PVC `data-postgres-0` (имя совпадает) — todo из hw6 остаются в базе. Проверить: `kubectl exec postgres-0 -- psql -U todo -d todo -tAc "select count(*) from todos;"`.

### Шаг 6. Установить ArgoCD

```text
# namespace + приватный age-ключ в кластер (repoServer им расшифровывает secrets://)
kubectl create namespace argocd
kubectl create secret generic helm-secrets-private-keys \
  --namespace argocd --from-file=key.txt=age-key.txt
 
# Helm-репозиторий и явная версия (не latest!)
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm search repo argo/argo-cd --versions | head -5
 
# Установка с нашим конфигом (проверено на chart 10.2.2 / app v3.4.6)
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 10.2.2 \
  --values argocd-values.yaml
 
kubectl -n argocd get pods -w
# argocd-repo-server стартует дольше всех (initContainer качает ~100 МБ) — дождись 1/1
```

`argocd-values.yaml` уже настроен: `repoServer` с плагином helm-secrets и age-ключом, сервисный аккаунт `github-actions` (для CI), Ingress `argocd.<домен>` с TLS, и переопределён реестр redis на Docker Hub (дефолтный `ecr-public.aws.com` недоступен с многих VPS → `ImagePullBackOff`). Дождись сертификата и зайди в UI:

```text
# Начальный пароль admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
 
# UI: https://argocd.<домен>  (login: admin / пароль выше)
# CLI login (server за ingress-nginx → grpc-web)
argocd login argocd.<домен> --username admin --grpc-web
```

### Шаг 7. Создать Application

Репо **публичный** (секреты зашифрованы SOPS) — ArgoCD читает его по HTTPS, никаких SSH deploy key не нужно. Просто применяем `Application`:

```text
kubectl apply -f application.yaml     # Application myapp (source: https://github.com/…/devops-gitops.git)
 
argocd app get myapp
# SyncStatus:   Synced
# HealthStatus: Healthy
```

ArgoCD задеплоит `myapp` (backend + frontend + ingress). `postgres` уже стоит (Шаг 5). Проверь, что приложение живо: `https://<домен>/` отдаёт фронт, `https://<домен>/api/health` → 200.

### Шаг 8. Security Context (проверка)

Security Context **уже в чарте** (`helm/myapp/templates/deployment-backend.yaml`): `runAsNonRoot`, `runAsUser: 1000`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, том `/tmp`. Твоя задача — убедиться, что backend поднялся под непривилегированным пользователем:

```text
POD=$(kubectl get pod -l app=backend -o name | head -1)
kubectl exec $POD -- id
# uid=1000 gid=1000 groups=1000
 
kubectl exec $POD -- touch /hack
# touch: /hack: Read-only file system   ← ожидаемо
kubectl exec $POD -- touch /tmp/ok      # /tmp writable (emptyDir)
```

> 💡 `runAsUser: 1000` обязателен: образ `node:20-alpine` без `USER` запускается от root, и `runAsNonRoot` без явного UID уронил бы Pod в `CreateContainerConfigError`.

### Шаг 9. Сервисный токен ArgoCD + секреты в code-репо

CI будет дёргать `argocd app sync` от имени сервисного аккаунта `github-actions` (он уже включён в `argocd-values.yaml`).

```text
argocd account generate-token --account github-actions
# eyJhbGciOiJIUzI1NiIs...   ← это ARGOCD_AUTH_TOKEN
```

В репозитории **`devops-hometask-01`** → Settings → Secrets and variables → Actions заведи:

### Шаг 10. GitOps-пайплайн в code-репо

Удали старый workflow hw6 и создай новый:

```text
git rm .github/workflows/deploy-k8s.yml
```

Создай `.github/workflows/deploy.yml` в `devops-hometask-01`. Он собирает образы, бампает теги в gitops-репо через PR и дёргает ArgoCD:

```yaml
name: CI/CD (GitOps)
 
on:
  push:
    branches: [week*]
 
env:
  IMAGE_TAG: sha-${{ github.sha }}
 
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ env.IMAGE_TAG }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASS }}
      - name: Build & push backend
        uses: docker/build-push-action@v6
        with:
          context: ./back
          push: true
          tags: ${{ secrets.DOCKER_USER }}/todo-back:${{ env.IMAGE_TAG }}
      - name: Build & push frontend
        uses: docker/build-push-action@v6
        with:
          context: ./front
          push: true
          build-args: VITE_API_URL=${{ vars.VITE_API_URL }}
          tags: ${{ secrets.DOCKER_USER }}/todo-front:${{ env.IMAGE_TAG }}
 
  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - name: Checkout gitops repo
        uses: actions/checkout@v4
        with:
          repository: ${{ github.repository_owner }}/devops-gitops
          token: ${{ secrets.GITOPS_TOKEN }}
          ref: main
          path: gitops
 
      - name: Install yq
        run: |
          sudo curl -sL https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -o /usr/local/bin/yq
          sudo chmod +x /usr/local/bin/yq
 
      - name: Bump image tags + PR + squash-merge
        env:
          GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}
          TAG: ${{ needs.build.outputs.image_tag }}
        run: |
          cd gitops
          yq e -i ".backend.image.tag = \"$TAG\""  helm/myapp/values-production.yaml
          yq e -i ".frontend.image.tag = \"$TAG\"" helm/myapp/values-production.yaml
          git config user.email "github-actions@github.com"
          git config user.name  "GitHub Actions"
          BRANCH="update-image-${{ github.run_id }}"
          git checkout -b "$BRANCH"
          git commit -am "chore: bump images to $TAG"
          git push origin "$BRANCH"
          REPO="${{ github.repository_owner }}/devops-gitops"
          PR=$(curl -sX POST -H "Authorization: Bearer $GITOPS_TOKEN" \
            https://api.github.com/repos/$REPO/pulls \
            -d "{\"title\":\"Update image to $TAG\",\"head\":\"$BRANCH\",\"base\":\"main\"}" \
            | python3 -c "import sys,json;print(json.load(sys.stdin)['number'])")
          curl -sX PUT -H "Authorization: Bearer $GITOPS_TOKEN" \
            https://api.github.com/repos/$REPO/pulls/$PR/merge \
            -d '{"merge_method":"squash"}'
 
      - name: ArgoCD sync
        env:
          ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}
          ARGOCD_SERVER: ${{ vars.ARGOCD_SERVER }}
        run: |
          sudo curl -sSL -o /usr/local/bin/argocd \
            https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
          sudo chmod +x /usr/local/bin/argocd
          argocd app sync myapp --grpc-web
          argocd app wait myapp --grpc-web --timeout 300
```

Запушь ветку — пайплайн прогонит всю цепочку:

```text
git push origin week7
```

Push в `week7` → `build` собирает образы `sha-<...>` → `deploy` бампает теги в gitops-репо через PR → `argocd app sync` применяет их в кластер. Проверь, что `/health` отдаёт свежий SHA:

```text
curl -s https://<домен>/api/health
# {"status":"ok","version":"sha-<коммит week7>"}
```

## Требования (Definition of Done)

- Приложение переведено на два Helm-чарта: `postgres` (вручную) и `myapp` (под ArgoCD). `helm lint` чистый.
- Секреты зашифрованы SOPS+age и лежат в **публичном** gitops-репо (значения `ENC[AES256_GCM,...]`); приватный `age-key.txt` не закоммичен; gitleaks-CI в gitops-репо зелёный.
- ArgoCD установлен, UI на `argocd.<домен>` с валидным TLS; `repoServer` расшифровывает `secrets://`.
- `Application myapp` — `Synced / Healthy`, `prune: true`, `selfHeal: true`.
- backend работает под `runAsUser: 1000` с read-only ФС.
- Push в `week7` собирает образы, бампает теги в gitops-репо через PR и триггерит `argocd app sync`.
- `https://<домен>/api/health` → `version` = SHA свежего коммита `week7`.
- `make e2e-deployed` зелёный.

## Проверка через e2e

Контракт домена не изменился (`/` → фронт, `/api` → бэк), поэтому e2e те же:

```text
# на локалке, в корне репо week7
make e2e-deployed
```

Если зелёные — сдавай ссылку `https://github.com/<логин>/devops-hometask-01/tree/week7` в форму. Домен агенту отдельно сообщать не надо — он берёт его из `.env.production.e2e`.

## Как проверяет AI-агент

Агент заходит по SSH на VPS (ключ `deploy` из hw3) и работает с кластером через `kubectl`; ArgoCD-объекты смотрит внутри кластера. Обязательные пункты:

**Наследие (из hw3):** SSH по ключу, `sshd -T` no-root/no-password, UFW (22/80/443 + 6443), fail2ban.

**Helm / postgres:**

- `helm list -A` содержит `postgres`; StatefulSet `1/1`, PVC `Bound`; данные переживают `kubectl delete pod postgres-0`.

**ArgoCD:**

- Поды `argocd` Running; Secret `helm-secrets-private-keys` есть; Ingress `argocd.<домен>` + Certificate `READY=True`.
- `Application myapp` — `Synced / Healthy`, source = публичный gitops-репо по HTTPS, `automated {prune, selfHeal}`.

**Секреты (gitops-репо публичный — агент читает напрямую):**

- gitops-репо публичный; в `helm/*/secrets.yaml` значения `ENC[AES256_GCM,...]` (не плейнтекст); `age-key.txt` **не** в репо; workflow `gitleaks` — последний ран зелёный.

**Приложение:**

- Deployment `backend`/`frontend` — доступные реплики = desired; Service `backend`(80→3000, порт `http`)/`frontend`; ConfigMap `backend-config`+`backend-appconfig`; Secret `backend-secret`.
- backend: `runAsUser: 1000`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`; `kubectl exec -- id` → uid=1000; запись в `/` запрещена.
- `https://<домен>/api/health` → 200, `version` = SHA свежего `week7`; `https://<домен>/` → фронт; TLS валиден.

**CI/CD (GitOps):**

- `.github/workflows/deploy.yml` есть, последний run на `week7` зелёный.
- Образы `todo-back:sha-<sha>`/`todo-front:sha-<sha>` на Docker Hub; в gitops-репо `values-production.yaml` содержит этот тег (коммит через смёрженный PR); backend в кластере бежит на этом образе.

**Регрессия:**

- `make e2e-deployed` зелёный (включая `metrics.spec.ts` через `/api/metrics`).

## Подсказки

- **`repoServer` не выходит в `1/1`.** initContainer качает ~100 МБ бинарников — подожди. Если `ImagePullBackOff`/OOM — проверь, что RAM апгрейднут (Шаг 0).
- **ArgoCD `ComparisonError: error unmarshaling ... secrets.yaml`.** Ключ age в кластере не тот, которым шифровали, либо `helm-secrets-private-keys` создан не в ns `argocd`. Пересоздай Secret тем же `age-key.txt`.
- **`Application` в `Degraded` из-за ServiceMonitor.** Ты включил `monitoring.enabled=true` без установленного `kube-prometheus-stack` (нет CRD). Оставь флаг `false` (это self-study).
- **`docker login` в CI падает `personal access token is expired`.** Токен `DOCKER_PASS` из прошлых ДЗ протух — пересоздай Docker Hub Access Token (Account → Security → New Access Token, Read/Write) и обнови секрет `DOCKER_PASS`.
- **`/health` отдаёт `version: latest`, а не `sha-...`.** ArgoCD синкнул образ `latest` из базовых `values`, а CI ещё не бампнул `values-production.yaml`. Проверь, что push в `week7` реально прошёл всю цепочку (Actions → зелёный `deploy`, PR в gitops-репо смёржен).
- **`argocd login`/`app sync` не коннектятся.** Server за ingress-nginx — добавляй `--grpc-web`. Проверь, что `ARGOCD_SERVER` = `argocd.<домен>` и Certificate `READY=True`.
- **`POST /todos` → 500.** Не смонтирован `backend-appconfig` (`/app/config/app.json`) — проверь, что ConfigMap `backend-appconfig` есть и volumeMount на месте (в чарте он уже прописан).
- **backend `0/1`, `CreateContainerConfigError: runAsNonRoot`.** В образе нет `USER`, а `runAsUser` не задан — в нашем чарте `runAsUser: 1000` есть; убедись, что не переопределил его пустым в `values-production.yaml`.

## Что дальше

В [self-study](https://it-incubator.io/react/ru/devops-for-devs/03-gitops-extras-self-study.mdx) — как убрать дублирование шаблонов через **library chart** в ghcr.io, добавить **DevSecOps**-сканы (Gitleaks/Semgrep/Trivy) в пайплайн и подключить второй сервер как **worker-ноду**.

---

# hw7 — self-study: library chart, DevSecOps, worker-нода

> Этот материал **не входит** в обязательную проверку. Сначала закрой ядро hw7 ([основная методичка](https://it-incubator.io/react/ru/devops-for-devs/02-helm-argocd-practice.mdx)): чарты, SOPS, ArgoCD, Security Context. Здесь — четыре независимых улучшения, которые можно делать в любом порядке и объёме.

## 1. Library chart в ghcr.io

Сейчас шаблоны `configmap.yaml`, `secret.yaml`, `deployment-*.yaml` — обычные файлы внутри чарта `myapp`. Если приложений станет несколько, эти шаблоны придётся копировать в каждый чарт и чинить баги во всех сразу. Решение — вынести общие шаблоны в **library chart**, опубликовать в **ghcr.io** как OCI-артефакт и подключать зависимостью.

### Создать library chart

```text
helm create common-chart
```

```text
apiVersion: v2
name: common-chart
description: Общие шаблоны для приложений
type: library        # library — нельзя деплоить напрямую, только через include
version: 0.1.0
```

Удали всё из `common-chart/templates/` и создай файлы, начинающиеся с `_` (Helm не рендерит их напрямую — только через `include`). Каждый оборачивается в `define`:

```text
{{- define "common-chart.backend-deployment" -}}
# ... тело backend-Deployment из helm/myapp/templates/deployment-backend.yaml ...
{{- end }}
```

Так же выноси `_configmap.yaml`, `_secret.yaml`, `_ingress.yaml`, `_hpa.yaml`, `_servicemonitor.yaml`.

### Опубликовать в ghcr.io

```text
echo $GITHUB_TOKEN | helm registry login ghcr.io --username ВАШ_ЛОГИН --password-stdin
helm package common-chart
helm push common-chart-0.1.0.tgz oci://ghcr.io/ВАШ_ЛОГИН
```

Сделай пакет публичным: GitHub → Packages → common-chart → Package settings → Change visibility.

### Подключить в чарт приложения

```yaml
dependencies:
  - name: common-chart
    version: "0.1.0"
    repository: "oci://ghcr.io/ВАШ_ЛОГИН"
```

Замени шаблоны в `helm/myapp/templates/` файлами-вызовами:

```text
{{ include "common-chart.backend-deployment" . }}
```

```text
helm dependency update helm/myapp     # создаст Chart.lock (коммить его)
echo "charts/" >> .gitignore          # сами .tgz не коммитим
```

ArgoCD перед каждым рендером делает `helm dependency build`, читает `Chart.lock` и тянет зависимость из ghcr.io — поэтому мы туда и публиковали. Новая версия library chart → бампаешь `version` в `Chart.yaml`, `helm package` + `helm push`, обновляешь версию зависимости и `Chart.lock`. ArgoCD подхватит.

## 2. DevSecOps: Gitleaks + Semgrep + Trivy

> **Gitleaks на gitops-репо — это уже hard-часть** (не self-study): раз gitops-репо публичный, `.github/workflows/gitleaks.yml` в нём страхует от случайно закоммиченного `age-key.txt` или плейнтекст-секрета. Он приезжает готовым в стартере. Здесь речь про **Semgrep + Trivy на code-репо** (и реюзбл-обвязку) — это остаётся self-study.

Три инструмента закрывают три класса проблем **до** билда:

- 🔑 **Gitleaks** — секреты/токены в git-истории;
- 🔍 **Semgrep** (SAST) — паттерны уязвимостей (OWASP Top 10, небезопасный код);
- 📦 **Trivy** (SCA) — CVE в зависимостях (`package-lock.json`).

`.gitleaks.toml` уже приехал в code-репо с патчем hw7 (`useDefault = true`). Оформи сканы как **реюзбл-воркфлоу** и вызывай первым джобом:

```yaml
name: SAST / SCA scan
on: workflow_call
jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          curl -sSL https://github.com/gitleaks/gitleaks/releases/download/v8.30.0/gitleaks_8.30.0_linux_x64.tar.gz -o g.tar.gz
          tar -zxf g.tar.gz && chmod +x gitleaks
          ./gitleaks git . --config=.gitleaks.toml --redact --no-banner --exit-code 1
  semgrep:
    runs-on: ubuntu-latest
    container: { image: semgrep/semgrep }
    steps:
      - uses: actions/checkout@v4
      - run: semgrep scan --config=p/typescript --config=p/nodejs --config=p/owasp-top-ten --error --exclude=node_modules --exclude=dist .
  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          curl -sL https://github.com/aquasecurity/trivy/releases/download/v0.69.2/trivy_0.69.2_Linux-64bit.tar.gz | tar -xz trivy
          ./trivy fs --scanners vuln --severity CRITICAL,HIGH --ignore-unfixed --exit-code 1 .
```

Вызов из основного пайплайна — первым, до `build` (сканам не нужен собранный артефакт):

```yaml
jobs:
  security-scan:
    uses: ./.github/workflows/sast-sca.yml
  build:
    needs: security-scan
    # ...
```

Рекомендуемый порядок подключения: сначала **Gitleaks** (разберись со старыми секретами в истории), потом **Trivy** (начни с `--severity CRITICAL`, если `HIGH` шумит), последним **Semgrep** (может давать false positives — глуши `# nosemgrep` там, где правило ложное; CVE без фикса — в `.trivyignore`).

## 3. Второй сервер как worker-нода

Если хочешь настоящий двухнодовый кластер (или у тебя простаивает старый VPS) — подключи его как k3s-agent. Это даёт больше ресурсов и базовую отказоустойчивость для stateless-подов.

```text
# на мастере — токен и IP
sudo cat /var/lib/rancher/k3s/server/node-token
curl -s ifconfig.me
ufw allow 6443/tcp     # если ещё не открыт
 
# на втором сервере (сначала погаси на нём всё лишнее: docker compose down)
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
 
# с мастера
kubectl get nodes
# old-vps   Ready   <none>   2m   v1.32.x
kubectl label node old-vps node-role.kubernetes.io/worker=worker
kubectl rollout restart deployment -n default   # перераспределить поды
```

Scheduler начнёт раскидывать новые поды по обеим нодам. Уже запущенные не переедут сами — их двигает `rollout restart`.

## 4. Security Context на frontend

В ядре hardened только backend — nginx капризен к `readOnlyRootFilesystem` (пишет в `/var/cache/nginx`, `/var/run`). Чтобы укрепить и фронт, нужен **unprivileged-образ nginx** (`nginxinc/nginx-unprivileged`, слушает 8080) или несколько `emptyDir`:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 101            # nginx в unprivileged-образе
containers:
  - name: frontend
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
    volumeMounts:
      - { name: cache, mountPath: /var/cache/nginx }
      - { name: run,   mountPath: /var/run }
volumes:
  - { name: cache, emptyDir: {} }
  - { name: run,   emptyDir: {} }
```

При смене образа/порта не забудь поправить `containerPort` и `targetPort` фронт-сервиса.

## 5. Включить ServiceMonitor и HPA

Если ты делал soft-мониторинг из hw6 (`kube-prometheus-stack`) и поставил metrics-server — включи флаги в `values-production.yaml`:

```yaml
monitoring:
  enabled: true     # появится ServiceMonitor (нужен CRD monitoring.coreos.com)
hpa:
  enabled: true     # появится HPA (нужен metrics-server)
```

Закоммить — ArgoCD применит. Без установленных зависимостей **не включай**: ServiceMonitor без CRD уведёт `Application` в `Degraded`.

## 6. Прямой push вместо PR (упрощение CI)

PR + squash-merge из основного пайплайна — production-практика, но хрупкая (скоупы токена, парсинг API). Для простоты можно пушить прямо в `main` gitops-репо:

```yaml
- name: Bump + push
  env: { GITOPS_TOKEN: ${{ secrets.GITOPS_TOKEN }}, TAG: ${{ needs.build.outputs.image_tag }} }
  run: |
    cd gitops
    yq e -i ".backend.image.tag = \"$TAG\""  helm/myapp/values-production.yaml
    yq e -i ".frontend.image.tag = \"$TAG\"" helm/myapp/values-production.yaml
    git config user.email "ci@github" && git config user.name "CI"
    git commit -am "chore: bump images to $TAG"
    git push origin main
```

Меньше движущихся частей, но теряешь историю через PR и защиту `main` от прямых пушей.