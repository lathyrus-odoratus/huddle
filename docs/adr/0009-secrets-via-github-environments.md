# ADR 0009：機密以 GitHub Environments secret 管理

- 狀態：已接受
- 日期：2026-10-03

## 背景

依 [ADR 0008](./0008-twelve-factor-adjustments.md)，應用設定與機密一律從環境注入。需要決定機密的資料來源。

真正的機密很少：`DISCORD_CLIENT_SECRET`、`WS_TICKET_SECRET`（各環境一份），加上 CI 用的 `CLOUDFLARE_API_TOKEN`。

Worker 在執行時不會去外部 vault 拉機密（會多一次網路請求和一個故障點），所以不管資料來源是什麼，最後都是在部署時寫入 Worker 的 secret。

## 考慮過的方案

- **GitHub OIDC 換 Cloudflare 短效 token**：Cloudflare 目前不支援信任外部 OIDC issuer（[workers-sdk Discussion #11434](https://github.com/cloudflare/workers-sdk/discussions/11434) 從 2025-11 至今沒有官方回應）。社群的 broker（cf-oidc-auth、cf-oidc-token-broker）需要一把權限更大的「能建立 API token 的母 token」，而且還在 1.0 之前，不採用。
- **Infisical / Doppler**：可以統一本機與 CI 的資料來源。但 Cloudflare token 仍然是長期 token，沒有安全性上的優勢，反而多一個外部依賴。機密數量少，不值得。
- **Vaultwarden**：自架的 Bitwarden 相容伺服器，用來管理「人」的密碼，不是給 CI 或應用程式用的 secret manager，不適用。
- **SOPS + age 加密放進 repo**：公開 repo，不採用。

## 決策

- 機密與各環境的應用設定存放在 **GitHub Environments**（`staging`、`production`）。
- CI 部署時寫入 Worker 的 secret / vars。
- 本機使用 `.dev.vars`（不進版控），只放 staging Discord App 的值，並提供 `.dev.vars.example`。
- Cloudflare API token：最小權限、設定 1 年到期並輪換、`production` Environment 需要人工核准。

## 影響

- 零額外服務，設定最少。
- 本機與 CI 是兩份資料，需要人工保持一致（本機只有 2 個值，可以接受）。
- 之後機密變多或有人加入協作時，可以換成 Infisical；因為程式碼只從環境讀取設定，所以只需要改 CI。
- Cloudflare 支援 OIDC 之後，應改用短效 token 並移除 `CLOUDFLARE_API_TOKEN`。
