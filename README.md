# devops-gitops

## Публикация Helm-чарта в OCI-реестр (GHCR)

```bash
echo $GITHUB_TOKEN | helm registry login ghcr.io --username ВАШ_ЛОГИН --password-stdin
helm package common-chart
helm push common-chart-0.1.0.tgz oci://ghcr.io/ВАШ_ЛОГИН
```

Три шага публикации чарта `common-chart` в GHCR как OCI-артефакта:

1. **`helm registry login`** — логин в `ghcr.io` по токену из `$GITHUB_TOKEN` (нужен PAT с правом `write:packages`).
2. **`helm package`** — собирает чарт из директории `common-chart` в архив `common-chart-0.1.0.tgz` (версия берётся из `Chart.yaml`).
3. **`helm push`** — заливает архив в GHCR под неймспейсом `ВАШ_ЛОГИН`; чарт становится доступен как `oci://ghcr.io/ВАШ_ЛОГИН/common-chart:0.1.0`.
