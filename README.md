# platform-gitops
Platform Engineering portfolio — GitOps management repository.

## 概要
ArgoCDのソースとなるリポジトリ。プラットフォーム全体の「あるべき姿」を管理する。

## ディレクトリ構成
| ディレクトリ | 内容 |
|---|---|
| `bootstrap/` | ArgoCDのRoot App（App of Apps） |
| `platform/` | 基盤コンポーネント（Ingress / Monitoring / Logging / Secrets / Policy） |
| `apps/` | サービス一覧 |

## Phase 2: GitOps & Secrets

### このPhaseで解決すること
ArgoCDによる「Gitの状態＝クラスタの状態」の実現と、
SecretをGitに直接置かないセキュアな基盤の構築。

### 構成コンポーネント
| コンポーネント | namespace | 役割 |
|---|---|---|
| ArgoCD | argocd | GitリポジトリをソースにK8sリソースを自動同期 |
| External Secrets Operator | external-secrets | 外部ストアからK8s Secretを自動生成 |

### ArgoCDへのアクセス
```bash
# 初期パスワードの取得
argocd admin initial-password -n argocd

# UIへのアクセス（port-forward）
kubectl port-forward svc/argocd-server -n argocd 8080:80
# → http://localhost:8080 (admin / 上記パスワード)

# CLIログイン
argocd login localhost:8080 --username admin --insecure
```

### App of Appsパターン
`bootstrap/root.yaml` を適用するだけで、`platform/` 以下の
全コンポーネントがArgoCDの管理下に入る。

```bash
kubectl apply -f bootstrap/root.yaml
argocd app list
```

### External Secrets Operatorの動作確認
```bash
# ClusterSecretStoreの確認
kubectl get clustersecretstore

# ExternalSecretの確認
kubectl get externalsecret -A
```

### 設計上の決定事項
- ArgoCDはHelmでインストールし、設定値を `platform/argocd/values.yaml` で管理。
- シークレットストアはローカル環境用に `Fake` プロバイダーを使用。Phase 8でAWS Secrets Managerに切り替え予定。
- PrivateリポジトリへのアクセスはSSH鍵で設定（`argocd repo add --ssh-private-key-path`）。
- APIバージョンは `external-secrets.io/v1`（v1beta1は非推奨）。

---

## Phase 3: Connectivity

### このPhaseで解決すること
Ingress-nginx と cert-manager を導入し、`*.localhost` で即座にHTTPSサービスを公開できる基盤を構築する。
自己署名CAによる証明書の自動発行により、開発者はTLS設定を意識せずにサービスを公開できる。

### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| ingress-nginx | ingress-nginx | ingress-nginx/ingress-nginx | 4.15.1 |
| cert-manager | cert-manager | jetstack/cert-manager | v1.20.2 |

### ClusterIssuerの構成
| リソース | 役割 |
|---|---|
| `selfsigned-issuer` | 自己署名でCA証明書を発行するIssuer |
| `local-ca`（Certificate） | selfsigned-issuerが発行したCA証明書。`cert-manager` namespaceの `local-ca-secret` に格納 |
| `local-ca-issuer` | `local-ca-secret` を使って各サービスの証明書を発行するIssuer |

### サービス公開の手順
Ingressマニフェストに以下のアノテーションを付与するだけで証明書が自動発行される。

```yaml
annotations:
  cert-manager.io/cluster-issuer: local-ca-issuer
```

### ArgoCDへのアクセス（Ingress経由）
Phase 3以降はport-forwardは不要。ブラウザから直接アクセスできる。

```
https://argocd.localhost
```

CLIログイン時にport-forwardが必要な場合：

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
argocd login localhost:8080 \
  --username admin \
  --password $(argocd admin initial-password -n argocd | head -1) \
  --insecure
```

### WindowsへのCA証明書インポート手順
クラスタ再作成後は新しいCA証明書をWindowsにインポートする必要がある。

```bash
# WSL: CA証明書をエクスポート
kubectl get secret local-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/local-ca.crt

cp /tmp/local-ca.crt \
  /mnt/c/Users/$(cmd.exe /c "echo %USERNAME%" 2>/dev/null | tr -d '\r')/Desktop/local-ca.crt
```

```powershell
# PowerShell（管理者）: 古い証明書を削除して再インポート
Get-ChildItem Cert:\LocalMachine\Root | Where-Object { $_.Subject -eq "CN=local-ca" } | Remove-Item
Import-Certificate -FilePath "$env:USERPROFILE\Desktop\local-ca.crt" -CertStoreLocation Cert:\LocalMachine\Root
```

### apps-rootについて
`bootstrap/apps-root.yaml` は `apps/` 直下の Application マニフェストのみを管理する（`recurse: false`）。
`apps/` 配下の実リソースは各 Application が管理するため、二重管理を防ぐ。

### 設計上の決定事項
- ingress-nginx の Admission Webhook（`ingress-nginx-admission`）は削除済み。k3dローカル環境では証明書検証エラーが発生するため。
- external-secrets の Application には `ServerSideApply=true` を設定。CRDのサイズ超過エラーを回避するため。
- ingress-nginx のリタイアについては Phase 8（EKS）で Gateway API への移行を検討する。移行先候補: Envoy Gateway、Cilium Gateway API。
