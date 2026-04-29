# platform-gitops

Platform Engineering portfolio — GitOps management repository.

## 概要

ArgoCDのソースとなるリポジトリ。プラットフォーム全体の「あるべき姿」を管理する。

## ディレクトリ構成

```
platform-gitops/
├── bootstrap/
│   ├── root.yaml                        # ArgoCD の Root App（App of Apps）
│   └── apps-root.yaml                   # サービス Application を管理する Root App
├── platform/
│   ├── argocd/                          # Phase 2
│   ├── ingress/                         # Phase 3
│   │   └── config/                      # Phase 10: CRD依存リソース（ClusterIssuer等）
│   ├── secrets/                         # Phase 2
│   │   └── config/                      # Phase 10: CRD依存リソース（ClusterSecretStore等）
│   ├── monitoring/                      # Phase 4
│   ├── logging/                         # Phase 4
│   ├── tracing/                         # Phase 4
│   ├── policy/                          # Phase 7
│   ├── cnpg/                            # Phase 8
│   ├── vpa/                             # Phase 9
│   └── goldilocks/                      # Phase 9
├── apps/
│   ├── sample-backend/                  # Phase 5/6
│   └── sample-frontend/                 # Phase 5/6
├── secrets/                             # SOPS × Age で暗号化した Secret（Phase 10）
├── .sops.yaml                           # SOPS 暗号化ルール
└── .mise.toml
```

---

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

# UIへのアクセス（Ingress経由 / Phase 3以降）
# https://argocd.localhost

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

### 設計上の決定事項
- ingress-nginx の Admission Webhook（`ingress-nginx-admission`）は削除済み。k3dローカル環境では証明書検証エラーが発生するため。
- external-secrets の Application には `ServerSideApply=true` を設定。CRDのサイズ超過エラーを回避するため。
- ingress-nginx のリタイアについては Phase 11（EKS）で Gateway API（Envoy Gateway）への移行を検討する。

---

## Phase 4: Observability

### このPhaseで解決すること
メトリクス・ログ・トレースの3本柱を揃え、サービスの内部状態を
「何が起きているか」を後から追跡できる基盤を構築する。

### 構成コンポーネント（LGTMスタック）
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| kube-prometheus-stack | monitoring | prometheus-community/kube-prometheus-stack | 83.6.0 |
| Loki | logging | grafana-community/loki | 6.55.0 |
| Grafana Alloy | logging | grafana/alloy | 1.7.0 |
| Tempo | tracing | grafana/tempo | 1.24.4 |

### アクセス
```
https://grafana.localhost
```

### 設計上の決定事項
- Grafana / Prometheus / Loki / Tempo を Grafana データソースとして統合し、ログ・メトリクス・トレースをひとつの画面で横断できる。
- `ServiceMonitor` を `common-app` Library Chart に組み込み、`serviceMonitor.enabled: true` だけでアプリの自動収集が完了する。
- Alloy（旧 Grafana Agent）でログ収集。Podラベルをベースにラベルを付与し、Lokiへ転送する。

---

## Phase 5: Platform Abstraction

### このPhaseで解決すること
アプリ開発者が Kubernetes マニフェストを直接書かなくてもデプロイできる「Golden Path」の入口を構築する。
`platform-charts` の Library Chart を `apps/` 配下の Application が参照する形で利用する。

### 構成コンポーネント
| Helmチャート | 役割 |
|---|---|
| `platform-charts/common-app` | Deployment / Service / Ingress / HPA / ServiceMonitor を一元管理 |
| `platform-charts/common-db` | CloudNativePG クラスタ / バックアップスケジュールを一元管理 |

### apps/ の構成
```
apps/
├── sample-backend/
│   ├── application.yaml    # ArgoCD Application リソース
│   └── values.yaml         # common-app + common-db の設定値
└── sample-frontend/
    ├── application.yaml
    └── values.yaml
```

---

## Phase 6: Golden Path（CI/CD）

### このPhaseで解決すること
アプリ Repo の push から ArgoCD による自動デプロイまでを、
開発者がプラットフォームを意識せずに完結できる CI/CD フローを構築する。

### フロー
```
push to main（sample-backend / sample-frontend）
  └─► GitHub Actions: build.yaml
        ├─ イメージビルド & GHCR プッシュ（タグ: git SHA short）
        └─► update-gitops.yaml
              └─► platform-gitops に repository_dispatch を送信
                    └─► .github/workflows/update-image.yaml
                          └─► apps/{service}/values.yaml の image.tag を更新して PR
                                └─► ArgoCD が自動同期
```

### 設計上の決定事項
- image tag の更新は PR 経由とし、デプロイの履歴を Git に残す。
- `GITOPS_TOKEN`（PAT）を GitHub Actions の Secret に登録し、クロスリポジトリの `repository_dispatch` を実現する。

---

## Phase 7: Guardrail（Kyverno）

### このPhaseで解決すること
ポリシーエンジン Kyverno によるガードレールの導入。
設定ミスやルール違反を「デプロイ前」に検出し、「誰が操作しても安全」な環境を実現する。

### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| Kyverno | kyverno | kyverno/kyverno | 3.7.1 |

### 適用ポリシー（`platform/policy/policies/`）
| ポリシー | 種別 | 内容 |
|---|---|---|
| `require-resource-limits.yaml` | Validate | CPU/Memoryの `limits` 未設定のPodをブロック |
| `disallow-latest-tag.yaml` | Validate | `image: xxx:latest` タグのPodをブロック |
| `add-default-labels.yaml` | Mutate | `app.kubernetes.io/name` ラベルを自動付与 |

### 設計上の決定事項
- ポリシー違反は ArgoCD の Sync 失敗として検出される。デプロイパイプラインの一部としてガードレールが機能する。
- `common-app` Library Chart は全ポリシーに準拠した値をデフォルトで持つため、Library Chart 経由でデプロイすれば自動的に通過する。

---

## Phase 8: Data & State

### このPhaseで解決すること
DB の「信頼できる状態」を管理する Library Chart を作り、
データの永続化・バックアップ・リストアをコード化する。

### 構成コンポーネント
| コンポーネント | namespace | Helmチャート | バージョン |
|---|---|---|---|
| CloudNativePG (CNPG) Operator | cnpg-system | cnpg/cloudnative-pg | 0.28.0 |

### 設計上の決定事項
- `common-db` Library Chart で CNPG Cluster を抽象化。`instances: 2` 以上でHA構成になる。
- バックアップ先はS3互換ストレージ（Phase 10で外部MinIOに移行）。
- `backup.enabled: true` にするだけで定期バックアップが有効になる。

---

## Phase 9: Resilience & Chaos

### このPhaseで解決すること
Nodeの障害時にサービスが継続できることを実際に障害を起こして検証する。

### 構成コンポーネント
| コンポーネント | namespace | 役割 |
|---|---|---|
| VPA | vpa | リソース上限の最適値を推定 |
| Goldilocks | goldilocks | VPA推定値をダッシュボードで可視化 |

### Chaos Engineering の検証
```bash
# ノード障害シミュレーション
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# CNPG フェイルオーバーの確認
kubectl get cluster -n <namespace> -w

# 復旧
kubectl uncordon <node-name>
```

### 設計上の決定事項
- `podAntiAffinity` を values に設定し、同一 Pod が同一 Node に集中しないことを保証する。
- Chaos Engineering の結果として DB Primary の切替（Switchover）の RTO を計測・記録した。

---

## Phase 10: DX & Secrets as Code

### このPhaseで解決すること
SecretのコードとしてのGit管理と、1コマンドで再現できるDR基盤の整備。

### SOPS × Age による Secrets as Code
Git 上の Secret は SOPS で暗号化し、Age キーを持つ開発者のみが復号できる。
`make bootstrap` 実行時に自動復号・適用されるため、手動のSecret投入作業がゼロになった。

```bash
# Secret の復号確認
sops --decrypt secrets/encrypted/minio-backup-secret.yaml
```

### Devcontainer によるオンボーディング最小化
`.devcontainer` と `mise` を組み合わせ、`git clone` 直後の環境から
標準化されたツールセットを最短で揃えられるようにする。

### 設計上の決定事項
- Age キーファイル（`~/.config/sops/age/keys.txt`）のみがGit管理外の唯一の前提条件。
- SOPS ルール（`.sops.yaml`）でファイルパスに応じた暗号化キーを自動選択する。
- `make bootstrap` に Secret の自動復号・投入を組み込み、手動作業をゼロにした。

---

## Phase 10.5: DR演習・sync-wave導入・外部MinIO移行

### このPhaseで解決すること
実際にクラスタを全消去し、完全自動復旧を実証する。
また演習を通じて発見した設計上の問題を修正し、より堅牢なプラットフォームに改善する。

### RTO計測結果

| 指標 | 時間 |
|---|---|
| RTO① `make bootstrap` 完了 | **7分37秒** |
| RTO② 全App Synced/Healthy | **15分24秒** |
| 手動作業 | 新規マシンの場合のAgeキーコピーのみ |

### ArgoCD sync-wave の導入

root appが管理する全Applicationにsync-waveを付与し、CRD依存の順序問題を構造的に解消した。

| wave | 対象App | 理由 |
|---|---|---|
| 1 | cert-manager, external-secrets, kyverno | CRD基盤。他のAppが依存するCRDを提供 |
| 2 | ingress-nginx, loki, alloy, tempo, vpa | CRD不要なコンポーネント |
| 3 | kube-prometheus-stack, goldilocks, argocd, kyverno-policies | ingress-nginx webhookが必要 |
| 4 | cnpg, cert-manager-config, external-secrets-config, sample-backend, sample-frontend | wave 3のCRD（PodMonitor等）が必要 |

### CRD依存リソースの分離

root appが直接管理するとCRDが存在しない状態でSyncが失敗する問題を解消するため、
CRD依存リソースを専用Applicationに分離した。

| Application | path | 管理するリソース |
|---|---|---|
| `cert-manager-config` | `platform/ingress/config/` | ClusterIssuer, Certificate |
| `external-secrets-config` | `platform/secrets/config/` | ClusterSecretStore, ClusterExternalSecret |

root appのexcludeにこれらのpathを追加し、直接管理を廃止した。

```yaml
# bootstrap/root.yaml
directory:
  recurse: true
  exclude: "{policy/policies/*,ingress/config/*,secrets/config/*}"
```

### 外部MinIOへの移行

クラスタ内MinIOではクラスタを消去するとバックアップも消滅するため、
WSL上のDockerコンテナとして外部MinIOを構築し、CNPGのバックアップ先を変更した。

```yaml
# apps/sample-backend/values.yaml
db:
  backup:
    endpointURL: "http://host.k3d.internal:9000"  # 外部MinIO
```

クラスタ内MinIO（`platform/minio/`）は廃止済み。

### 設計上の決定事項
- sync-wave導入により、`make bootstrap` 実行後は完全自動でSynced/Healthyになる。
- Kyvernoの `resources.limits` 必須ポリシーがCNPG recoveryをブロックする問題を発見。recoveryマニフェストには必ずresourcesを明示する。
- クラスタ内バックアップストレージはDR対象外になるため、本番・検証環境では必ずクラスタ外に置く。
- Phase 13（EKS移行）でバックアップ先をAWS S3に移行し、真のクラウドDRを実現予定。
