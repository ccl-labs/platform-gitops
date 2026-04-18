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
\`\`\`bash
# 初期パスワードの取得
argocd admin initial-password -n argocd

# UIへのアクセス（port-forward）
kubectl port-forward svc/argocd-server -n argocd 8080:80
# → http://localhost:8080 (admin / 上記パスワード)

# CLIログイン
argocd login localhost:8080 --username admin --insecure
\`\`\`

### App of Appsパターン
\`bootstrap/root.yaml\` を適用するだけで、\`platform/\` 以下の
全コンポーネントがArgoCDの管理下に入る。

\`\`\`bash
kubectl apply -f bootstrap/root.yaml
argocd app list
\`\`\`

### External Secrets Operatorの動作確認
\`\`\`bash
# ClusterSecretStoreの確認
kubectl get clustersecretstore

# ExternalSecretの確認
kubectl get externalsecret -A
\`\`\`

### 設計上の決定事項
- ArgoCDはHelmでインストールし、設定値を \`platform/argocd/values.yaml\` で管理。
- シークレットストアはローカル環境用に \`Fake\` プロバイダーを使用。Phase 8でAWS Secrets Managerに切り替え予定。
- PrivateリポジトリへのアクセスはSSH鍵で設定（\`argocd repo add --ssh-private-key-path\`）。
- APIバージョンは \`external-secrets.io/v1\`（v1beta1は非推奨）。
