# platform-cicd-workflows

GitHub Actions **reutilizáveis** da plataforma de dados event-driven, com **CI** (validar, testar, construir) e **CD** (aplicar, implantar, promover, versionar).

Os repositórios da plataforma e dos domínios **não escrevem pipelines**: eles chamam estes workflows e fixam a tag major `@v1`.

## Catálogo

| Workflow | Tipo | O que faz | Usado por |
|---|---|---|---|
| `ci-terraform.yml` | CI | `fmt`, `validate` e `plan` com OIDC, com o plan comentado no PR | `platform-infra` |
| `cd-terraform.yml` | CD | `plan` + `apply` com OIDC, protegido por GitHub Environment; `destroy: true` desliga ambientes efêmeros | `platform-infra` |
| `ci-container.yml` | CI | `docker build`, scan Trivy e push opcional ao ACR (tag `X.Y.Z` ou `sha-xxxxxxx`) | `*-service` |
| `cd-gitops-promote.yml` | CD | Atualiza a versão da imagem no `platform-gitops` (commit direto ou PR); o Flux reconcilia | `*-service` |
| `ci-python.yml` | CI | `ruff`, `pytest` e build do wheel | SDK, frameworks |
| `cd-python-release.yml` | CD | Na tag `vX.Y.Z`, valida o `pyproject`, publica o wheel como asset de GitHub Release | SDK, frameworks |
| `cd-major-tag.yml` | CD | Move a tag major (`v1`) para a última `v1.Y.Z` | Repositórios consumidos como dependência |
| `ci-databricks-bundle.yml` | CI | `databricks bundle validate` com OIDC | `*-data` |
| `cd-databricks-bundle.yml` | CD | `databricks bundle deploy` (+ `run` opcional), protegido por Environment | `*-data` |
| `ci-kustomize.yml` | CI | `kustomize build` + `kubeconform` (com CRDs do Flux) | `platform-gitops` |
| `ci-contracts.yml` | CI | `edp-contracts`: lint ODCS v3.2.0 + regras da plataforma, schemas gerados em dia e, em PR, bloqueio de breaking changes vs branch base | `*-service` |

## Fluxo CI → CD por tipo de componente

```
platform-infra      PR ─▶ ci-terraform (plan no PR)        merge ─▶ cd-terraform (apply, env dev)
<dominio>-service   PR ─▶ ci-contracts + ci-container (build+scan)        merge ─▶ ci-container (push) ─▶ cd-gitops-promote (dev)
                                                             tag  ─▶ cd-gitops-promote (prd, mode=pull-request)
<dominio>-data      PR ─▶ ci-databricks-bundle (validate)  merge ─▶ cd-databricks-bundle (deploy dev)
frameworks / SDK    PR ─▶ ci-python                        tag  ─▶ cd-python-release + cd-major-tag
platform-gitops     PR ─▶ ci-kustomize                     merge ─▶ (Flux reconcilia no cluster: GitOps pull)
```

## Pré-requisitos no repositório chamador

| Item | Onde | Para quê |
|---|---|---|
| `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` | **Variables** (não secrets) | Login OIDC; são IDs, não credenciais |
| Federated credential na App Registration | Entra ID (criada pelo `platform-infra/bootstrap`) | Subjects `repo:<owner>/<repo>:pull_request` e `repo:<owner>/<repo>:environment:<env>` |
| GitHub Environment `dev` (e `prd`) | Settings → Environments | Gate de aprovação do CD + subject do OIDC |
| `GITOPS_DEPLOY_KEY` ou `GITOPS_TOKEN` | **Secret** (só nos `*-service`) | Deploy key SSH com escrita só no `platform-gitops` (basta para `mode=commit`) ou PAT/GitHub App (necessário para `mode=pull-request`) |
| `permissions: id-token: write` | No workflow chamador | Obrigatório para OIDC |

> As variáveis do Azure são passadas como **inputs**, e não lidas via `vars` dentro do workflow reutilizável. Isso deixa explícito o que cada pipeline usa e permite que o CI rode só a validação (sem plan) enquanto o Azure não estiver configurado.

## Exemplos

Veja [`examples/`](examples/):
- [`terraform-caller.yml`](examples/terraform-caller.yml)
- [`service-caller.yml`](examples/service-caller.yml)
- [`python-lib-caller.yml`](examples/python-lib-caller.yml)
- [`bundle-caller.yml`](examples/bundle-caller.yml)

## Versionamento

- SemVer: `vX.Y.Z`. O push de uma tag dispara o `release.yml`, que cria a GitHub Release e move a tag `vX`.
- Uma mudança **breaking** em inputs ou outputs exige um novo major (`v2`).
- O CI deste repositório (`self-lint.yml`) roda o `actionlint` em todo PR.

## ADRs relacionados

ADR-0007 (Polyrepo + GitOps), ADR-0006 (orquestração Databricks), ADR-0005 (contratos). Ver `platform-docs/adr` (local).
