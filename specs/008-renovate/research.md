# Research: Renovate導入

**Date**: 2026-01-03  
**Feature**: 008-renovate

## 1. Renovate Manager選択

### Decision: Flux Manager + GitHub Actions Manager

**Rationale**: 
- FluxCDのHelmReleaseは`flux` managerがネイティブサポート
- GitHub Actionsは`github-actions` managerがネイティブサポート
- 両方を組み合わせることで、インフラとCIの両方をカバー

**Alternatives Considered**:
- `helm-values` manager: HelmRelease内のimageタグ更新に使用可能だが、今回はchartバージョンのみが対象なので不要
- `regex` manager: カスタムパターンに対応できるが、標準managerで十分

## 2. FluxCD HelmRelease更新設定

### Decision: managerFilePatterns でk8s/infrastructure配下を対象化

**Rationale**:
- デフォルトでは`gotk-components.yaml`のみが対象
- `k8s/infrastructure/**/*.yaml`を対象に含めることで全HelmReleaseをカバー
- `spec.chart.spec.version`フィールドを自動検出・更新

**Configuration**:
```json
{
  "flux": {
    "fileMatch": ["k8s/infrastructure/.+\\.yaml$"]
  }
}
```

**Alternatives Considered**:
- 各HelmReleaseファイルを個別指定: 保守性が低いため却下
- `k8s/**/*.yaml`を全対象: 不要なファイルもスキャンされるため範囲を限定

## 3. GitHub Actions更新設定

### Decision: helpers:pinGitHubActionDigests プリセット使用

**Rationale**:
- セキュリティ向上のため、アクションをSHAにピン留め
- バージョンタグとSHAの両方を管理
- `config:recommended`プリセットに含まれる標準設定を活用

**Configuration**:
```json
{
  "extends": [
    "config:recommended",
    "helpers:pinGitHubActionDigests"
  ]
}
```

**Alternatives Considered**:
- SHAピン留めなし: サプライチェーン攻撃のリスクがあるため却下
- 手動でのバージョン固定: 自動化の利点が失われるため却下

## 4. SOPS暗号化ファイルの除外

### Decision: ignorePaths で `*.sops.yaml` パターンを除外

**Rationale**:
- SOPS暗号化ファイルはRenovateが解析できない
- 誤った更新PRを防止
- globパターンで一括除外が可能

**Configuration**:
```json
{
  "ignorePaths": [
    "**/*.sops.yaml",
    ".sops.yaml"
  ]
}
```

**対象ファイル（7個）**:
- `.sops.yaml`（ルート設定）
- `infra/talos/talenv.sops.yaml`
- `infra/talos/talsecret.sops.yaml`
- `k8s/infrastructure/tailscale/secret-oauth-credentials.sops.yaml`
- `k8s/infrastructure/grafana-alloy/secret-grafana-cloud.sops.yaml`
- `k8s/infrastructure/longhorn/secret-r2-credentials.sops.yaml`
- `k8s/infrastructure/longhorn/examples/secret-demo.sops.yaml`

**Alternatives Considered**:
- 個別ファイル指定: 新規SOPSファイル追加時に設定漏れのリスク
- `enabled: false`でmanager無効化: 他のYAMLファイルも対象外になるため不適切

## 5. Flux自動生成ファイルの除外

### Decision: ignorePaths で gotk-components.yaml を除外

**Rationale**:
- `gotk-components.yaml`はFlux CLIが自動生成
- 手動編集やRenovate更新は想定されていない
- Fluxのアップグレードは別途手動で実施

**Configuration**:
```json
{
  "ignorePaths": [
    "k8s/flux/flux-system/gotk-components.yaml"
  ]
}
```

**Alternatives Considered**:
- Flux manager のデフォルト対象のまま: 意図しない更新PRが作成されるリスク
- `flux-system`ディレクトリ全体除外: `gotk-sync.yaml`も除外されるが問題なし

## 6. PR制限とグループ化戦略

### Decision: prConcurrentLimit: 10、separateMajorMinor: true

**Rationale**:
- 仕様で同時オープンPR数を10個に制限と決定済み
- メジャーバージョン更新は個別PRで慎重にレビュー
- マイナー/パッチはグループ化可能だが、今回は個別PRを維持

**Configuration**:
```json
{
  "prConcurrentLimit": 10,
  "separateMajorMinor": true,
  "separateMultipleMajor": true
}
```

**Alternatives Considered**:
- グループ化（`groupName`）: 変更の影響範囲が把握しづらくなるため、当面は個別PR維持
- prConcurrentLimit: 5: 積極的な更新方針のため10を選択

## 7. スケジュール設定

### Decision: デフォルトスケジュール（随時実行）

**Rationale**:
- GitHub Appはスケジュールを自動管理
- 新バージョン検出から24時間以内のPR作成要件を満たす
- 特定時間帯に制限する必要なし（Homelabは常時稼働）

**Configuration**:
```json
{
  "schedule": ["at any time"]
}
```

**Alternatives Considered**:
- 週末のみ実行: 更新の遅延が発生するため却下
- 深夜のみ実行: Homelabでは不要な制約

## 8. 自動マージ設定

### Decision: automerge: false（全PR手動レビュー）

**Rationale**:
- 仕様で「自動マージは行わず、すべてのPRは手動レビュー後にマージ」と決定済み
- Homelabとはいえ、本番環境への影響を慎重に確認

**Configuration**:
```json
{
  "automerge": false
}
```

**Alternatives Considered**:
- パッチバージョンのみ自動マージ: 仕様の方針に反するため却下
- CI成功時のみ自動マージ: 同上

## 9. ベースプリセット選択

### Decision: config:recommended を拡張

**Rationale**:
- Renovateの推奨設定を基盤として使用
- プロジェクト固有の設定のみオーバーライド
- メンテナンス性が高い

**Configuration**:
```json
{
  "extends": [
    "config:recommended",
    ":disableRateLimiting"
  ]
}
```

**Alternatives Considered**:
- `config:base`: 設定が最小限すぎる
- プリセットなし: すべてを手動設定する必要があり、保守性が低い

## Summary: 最終設定構成

```json5
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    "helpers:pinGitHubActionDigests"
  ],
  
  // PR制限
  "prConcurrentLimit": 10,
  "separateMajorMinor": true,
  "separateMultipleMajor": true,
  "automerge": false,
  
  // 除外パス
  "ignorePaths": [
    "**/*.sops.yaml",
    ".sops.yaml",
    "k8s/flux/flux-system/gotk-components.yaml"
  ],
  
  // Flux HelmRelease対象
  "flux": {
    "fileMatch": ["k8s/infrastructure/.+\\.yaml$"]
  },
  
  // ラベル設定
  "labels": ["dependencies", "renovate"]
}
```
