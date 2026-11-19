# gitops-operator-release

Build + publish image [`gitops-operator`](https://github.com/redhat-developer/gitops-operator)
(redhat-developer, OpenShift GitOps operator) lên GHCR cho các cluster
passiontech-k0s — upstream **không publish image public** (quay repo private,
`registry.redhat.io` cần credentials Red Hat), nên release repo này build verbatim
từ source tag upstream (không sửa source).

## Image

```
ghcr.io/passiontech-global/gitops-operator:<upstream-tag>   # vd v1.20.0
ghcr.io/passiontech-global/gitops-operator:latest
```

Package visibility phải là **public** để node k0s pull anonymous (org → Packages
→ `gitops-operator` → Package settings → Change visibility).

## Cách hoạt động

- Workflow `.github/workflows/build.yml` chạy 1 lần/ngày (cron `17 3 * * *`):
  so tag mới nhất upstream (`releases/latest` — tự loại RC/draft) với file
  `VERSION`; khác → build + push + cập nhật `VERSION`. Giống nhau → skip.
- Trigger tay: tab Actions → `build-operator-image` → Run workflow → nhập
  `tag` (trống = latest).
- Build chỉ publish image. Việc deploy lên cluster do GitOps
  (`passiontech-global/passiontech-fluxcd`) quyết định qua renovate/PR bump pin
  image trong `clusters/_shared/gitops-operator.yaml`.

## Cắt conveyor vào fluxcd

`versions.yaml` + `clusters/_shared/gitops-operator.yaml` trong
passiontech-fluxcd pin image này; renovate theo dõi
`ghcr.io/passiontech-global/gitops-operator` để mở PR khi có tag mới.

## Quy trình bump version (theo đúng thứ tự)

1. **Check upstream** có tag mới chưa:
   ```
   gh api repos/redhat-developer/gitops-operator/releases/latest --jq .tag_name
   ```
   So với file `VERSION` của repo này — giống → không có gì để làm.
2. **Trigger build** trong repo này (GHCR phải có tag trước khi fluxcd bump
   pin — build chỉ publish lên GHCR, không deploy):
   ```
   gh workflow run build.yml -R passiontech-global/gitops-operator-release --ref main
   # hoac tag cu the:
   gh workflow run build.yml -R passiontech-global/gitops-operator-release -f tag=vX.Y.Z
   gh run watch <run-id> -R passiontech-global/gitops-operator-release
   ```
   Run xong → workflow tự commit `VERSION` mới vào repo này.
3. **Bump pin** trong `passiontech-fluxcd`:
   `clusters/_shared/gitops-operator.yaml` (image pin) — renovate tự mở PR
   khi GHCR có tag mới, merge PR = deploy lên cả 3 cluster (KS health gate
   chặn nếu operator lỗi).
