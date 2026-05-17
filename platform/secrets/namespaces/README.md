# secrets/namespaces

`external-secrets-config` App（wave 12）が ExternalSecret を keycloak・backstage・monitoring namespace へデプロイするため、それらの namespace を事前に作成する。

## なぜ明示的に書いているか

ArgoCD の `CreateNamespace=true` は `spec.destination.namespace` の1つしか作成しない。
`external-secrets-config` の destination は `argocd` であり、他 namespace への ExternalSecret デプロイ時にそれらが存在しないと apply が失敗する。

各 namespace を本来作成する App（keycloak-db: wave 13、backstage-db: wave 21 等）は wave 12 より後に動くため、ここで先行して作成している。

## 対象外の namespace

上記以外の namespace は各 App の `CreateNamespace=true` で自動作成される。
このディレクトリはすべての namespace を網羅するものではなく、wave 12 の ESO デプロイに必要なものだけを置く。
