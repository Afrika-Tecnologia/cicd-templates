# 🔐 CI/CD Templates — Security Gate

Repositório contendo o workflow reusável de **Security Gate** para pipelines CI/CD, integrando scans de segurança via **Veracode** (Pipeline Scan, SAST, SCA e IaC).

---

## Visão Geral

Este repositório disponibiliza um único workflow reusável (`ci_security_scans.yaml`) que implementa um **gate de segurança em 3 tiers**, decidindo automaticamente quais scans executar com base no contexto de branch e evento (push vs. pull request).

```
┌─────────────────────────────────────┐      ┌────────────────────────────────────┐
│  Este repo (templates)              │      │  Repo da aplicação                 │
│                                     │      │                                    │
│  .github/workflows/                 │◄─────│  .github/workflows/                │
│    ci_security_scans.yaml           │ uses │    security.yaml (caller)          │
└─────────────────────────────────────┘      └────────────────────────────────────┘
```

---

## Tiers de Segurança

| Tier | Trigger | Scans executados | Bloqueante? |
|------|---------|------------------|-------------|
| **Tier 1** | Push / Commit | Pipeline Scan + SCA + IaC | Não |
| **Tier 2** | PR → branch staging (`develop`, `staging`, etc.) | Pipeline Scan + SAST (sandbox) + SCA + IaC | Não |
| **Tier 3** | PR → branch production (`main`, `master`) | Pipeline Scan + SAST (policy scan) + SCA + IaC | **Sim** |

> O tier é determinado automaticamente pelo job `determine_tier` com base na branch-alvo do PR e nos patterns configurados.

---

## Como Usar

### 1. Configurar Secrets no repositório da aplicação

Vá em **Settings → Secrets and variables → Actions** no repo da sua aplicação e adicione:

| Secret | Descrição | Obrigatório |
|--------|-----------|:-----------:|
| `SCAN_API_TOKEN` | Veracode SCA API Token | ✅ |
| `SCAN_API_ID` | Veracode API ID | ✅ |
| `SCAN_API_KEY` | Veracode API Key | ✅ |
| `BANTUU_API_KEY` | Bantuu API Key (Pipeline Scan e SAST via Veracode-Connect) | Opcional |

### 2. Criar o workflow caller na aplicação

Crie o arquivo `.github/workflows/security.yaml` no repositório da sua aplicação:

```yaml
name: Security Scans

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:

jobs:
  security:
    uses: <org>/<repo>/.github/workflows/ci_security_scans.yaml@main
    with:
      runner: ubuntu-latest
      # scan_path: '.'                              # (default) Diretório a escanear
      # build_artifact_name: ''                     # Vazio = scan do source code
      # veracode_policy_name: 'Security Policy - Nivel 2'
      # staging_branch_patterns: 'develop,dev,staging,qa,homolog'
      # production_branch_patterns: 'main,master'
    secrets:
      SCAN_API_TOKEN: ${{ secrets.SCAN_API_TOKEN }}
      SCAN_API_ID: ${{ secrets.SCAN_API_ID }}
      SCAN_API_KEY: ${{ secrets.SCAN_API_KEY }}
      BANTUU_API_KEY: ${{ secrets.BANTUU_API_KEY }}
```

> Substitua `<org>/<repo>` pelo caminho real deste repositório de templates (ex: `minha-org/cicd-templates`).

---

## Inputs

| Input | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `runner` | `string` | `ubuntu-latest` | Runner label para os jobs |
| `scan_path` | `string` | `.` | Diretório do repositório a ser escaneado |
| `build_artifact_name` | `string` | `''` | Nome do artefato de build. Se vazio, escaneia o source code |
| `build_artifact_path` | `string` | `build` | Path onde o artefato de build será baixado/extraído |
| `build_artifact_exclude_patterns` | `string` | `''` | Patterns de exclusão separados por vírgula (ex: código de teste) |
| `project_name` | `string` | `''` | Nome do projeto. Se vazio, usa o nome do repositório |
| `artifact_suffix` | `string` | `''` | Sufixo para artefatos (útil em monorepos, ex: `-erp`, `-lms`) |
| `veracode_policy_name` | `string` | `Security Policy - Nivel 2` | Nome da policy Veracode para o Pipeline Scan |
| `staging_branch_patterns` | `string` | `develop,dev,stage,staging,qa,homolog,homologacao` | Branches consideradas staging (Tier 2) |
| `production_branch_patterns` | `string` | `main,master` | Branches consideradas production (Tier 3) |

---

## Jobs do Workflow

| Job | Descrição | Condição |
|-----|-----------|----------|
| **determine_tier** | Analisa o contexto (branch/PR) e define o tier de segurança | Sempre |
| **autopackager** | Empacota o código/artefato para análise Veracode | Sempre |
| **sca** | Veracode Software Composition Analysis (dependências) | Sempre |
| **iac** | Veracode Infrastructure as Code scan | Sempre |
| **pipeline_scan** | Veracode Pipeline Scan (análise estática rápida) | Sempre (após autopackager) |
| **sast** | Veracode Upload & SAST (análise estática completa) | Apenas Tier 2 e 3 |

---

## Uso com Build Artifact

Para projetos que precisam compilar antes do scan (ex: Java com JAR), passe o artefato de build:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      - run: mvn package -DskipTests
      - uses: actions/upload-artifact@v4
        with:
          name: java-build
          path: target/

  security:
    needs: [build]
    uses: <org>/<repo>/.github/workflows/ci_security_scans.yaml@main
    with:
      build_artifact_name: 'java-build'
      build_artifact_path: 'build'
      runner: ubuntu-latest
    secrets:
      SCAN_API_TOKEN: ${{ secrets.SCAN_API_TOKEN }}
      SCAN_API_ID: ${{ secrets.SCAN_API_ID }}
      SCAN_API_KEY: ${{ secrets.SCAN_API_KEY }}
      BANTUU_API_KEY: ${{ secrets.BANTUU_API_KEY }}
```

---

## Uso em Monorepos

Para monorepos com múltiplos projetos, use `scan_path` e `artifact_suffix`:

```yaml
jobs:
  security-erp:
    uses: <org>/<repo>/.github/workflows/ci_security_scans.yaml@main
    with:
      scan_path: 'packages/erp'
      artifact_suffix: '-erp'
    secrets: # ...

  security-lms:
    uses: <org>/<repo>/.github/workflows/ci_security_scans.yaml@main
    with:
      scan_path: 'packages/lms'
      artifact_suffix: '-lms'
    secrets: # ...
```

---

## Visibilidade do Repositório

| Visibilidade deste repo | Quem pode chamar o workflow |
|--------------------------|------------------------------|
| **Public** | Qualquer repo de qualquer organização |
| **Internal** | Repos da mesma organização (Enterprise) |
| **Private** | Somente repos da **mesma organização** |

---

## Troubleshooting

| Problema | Solução |
|----------|---------|
| `not found` ao referenciar o workflow | Verifique se este repo é público ou se ambos estão na mesma org |
| Erro de autenticação Veracode | Confirme que os secrets estão no repo **caller** (aplicação) |
| Action `Veracode-Connect` não encontrada | A action precisa ser pública ou copiada para sua org |
| SAST não executa | Confira se é um **PR** (não push) contra uma branch staging ou production |
| Auto Packager falha | Verifique se o código/artefato está no path correto |

---

## Documentação Adicional

- [Guia de Teste em Ambiente Próprio](./docs/testing_ci_security_scans.md)
- [GitHub Docs — Reusing Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Veracode Documentation](https://docs.veracode.com/)
