# Contract: Renovate Reconciliation

**Date**: 2026-01-03  
**Feature**: 008-renovate

## Overview

このドキュメントはRenovateの動作契約を定義する。Renovateがどのように依存関係を検出し、PRを作成するかの期待動作を記述する。

## Contract 1: HelmRelease Version Detection

### Trigger
- Renovateスケジュール実行時
- HelmRepositoryに新バージョンが存在

### Input
```yaml
# k8s/infrastructure/*/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
spec:
  chart:
    spec:
      chart: <chart-name>
      version: <current-version>
      sourceRef:
        kind: HelmRepository
        name: <repo-name>
```

### Expected Output
- 新バージョンが存在する場合: PR作成
- 新バージョンがない場合: 何もしない
- 対象ファイルが`ignorePaths`に該当: スキップ

### PR Content
```
Title: chore(deps): update helm release <chart-name> to <new-version>
Branch: renovate/<chart-name>-<major>.x
Labels: dependencies, renovate
Body: 
  - Version change: <old> → <new>
  - Release notes link
  - Changelog excerpt (if available)
```

### Validation
- [ ] 6つのHelmReleaseすべてが検出対象
- [ ] バージョン更新がPRに正しく反映
- [ ] リリースノートリンクが含まれる

---

## Contract 2: GitHub Actions Version Detection

### Trigger
- Renovateスケジュール実行時
- アクションに新バージョンが存在

### Input
```yaml
# .github/workflows/*.yaml
steps:
  - uses: actions/checkout@v4
  - uses: aquasecurity/trivy-action@0.28.0
```

### Expected Output
- 新バージョンが存在する場合: PR作成
- SHAピン留めが必要な場合: コミットSHAも更新
- 新バージョンがない場合: 何もしない

### PR Content
```
Title: chore(deps): update actions/checkout action to v5
Branch: renovate/actions-checkout-5.x
Labels: dependencies, renovate
Body:
  - Version change with SHA
  - Release notes link
```

### Validation
- [ ] ワークフロー内の全アクションが検出
- [ ] SHAピン留めが適用される
- [ ] バージョン更新がPRに正しく反映

---

## Contract 3: SOPS File Exclusion

### Trigger
- Renovateがファイルをスキャン

### Input
```
任意の *.sops.yaml ファイル
```

### Expected Output
- ファイルは処理対象から除外
- PRは作成されない
- エラーログなし

### Validation
- [ ] 7つのSOPSファイルすべてがスキップ
- [ ] 除外ファイルに対するPRが0件

---

## Contract 4: Flux Generated File Exclusion

### Trigger
- Renovateがファイルをスキャン

### Input
```
k8s/flux/flux-system/gotk-components.yaml
```

### Expected Output
- ファイルは処理対象から除外
- PRは作成されない

### Validation
- [ ] gotk-components.yamlがスキップ
- [ ] Flux自動生成ファイルに対するPRが0件

---

## Contract 5: PR Concurrency Limit

### Trigger
- 10個以上の更新が同時に検出

### Expected Behavior
- 最大10個のPRのみ作成
- 優先度の高い更新が先に処理
- 残りは次回実行時に処理

### Validation
- [ ] 同時オープンPRが10個を超えない
- [ ] 既存PRがマージ/クローズ後に新PRが作成

---

## Contract 6: Major Version Separation

### Trigger
- メジャーバージョン更新が検出（例: v1.x → v2.x）

### Expected Behavior
- メジャー更新は個別PRで作成
- マイナー/パッチ更新とグループ化しない
- 複数のメジャー更新がある場合も各々別PR

### PR Naming
```
Major update: renovate/<package>-<major>.x
Minor update: renovate/<package>-<major>.x (same branch updated)
```

### Validation
- [ ] メジャー更新が個別PRで作成
- [ ] マイナー/パッチとグループ化されない

---

## Contract 7: No Automerge

### Trigger
- PRが作成される

### Expected Behavior
- すべてのPRは`Open`状態で待機
- 自動マージは発生しない
- 手動レビュー・マージが必要

### Validation
- [ ] automerge設定がfalse
- [ ] PRが自動的にマージされない

---

## Error Handling

### Invalid Configuration
- `renovate.json5`に構文エラーがある場合
- → Renovate Dashboardにエラー表示
- → PRは作成されない

### Network Failure
- レジストリへのアクセスが失敗した場合
- → Renovateは次回実行時にリトライ
- → 一時的な障害は自動回復

### Rate Limiting
- GitHub APIのレート制限に達した場合
- → Renovateは待機後にリトライ
- → `prConcurrentLimit`で緩和
