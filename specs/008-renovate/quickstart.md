# Quickstart: Renovate導入

**Date**: 2026-01-03  
**Feature**: 008-renovate

## Prerequisites

- GitHubリポジトリへのAdmin権限
- GitHub Renovate Appのインストール権限

## Step 1: GitHub Renovate Appのインストール

1. [Renovate GitHub App](https://github.com/apps/renovate)にアクセス
2. "Install" をクリック
3. `aoshimash/homelab-k8s` リポジトリを選択
4. 権限を確認して承認

## Step 2: 設定ファイルの作成

リポジトリルートに `renovate.json5` を作成:

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

## Step 3: 初回実行の確認

1. `renovate.json5`をコミット・プッシュ
2. RenovateがOnboarding PRを作成（設定確認用）
3. PRの内容を確認:
   - 検出された依存関係一覧
   - 適用される設定
4. Onboarding PRをマージ

## Step 4: 動作確認

### 検出される依存関係の確認

Renovate Dashboardで以下が検出されることを確認:

**HelmReleases (6個)**:
- [ ] cilium
- [ ] longhorn
- [ ] tailscale-operator
- [ ] grafana-alloy
- [ ] kube-state-metrics
- [ ] metrics-server

**GitHub Actions**:
- [ ] actions/checkout
- [ ] aquasecurity/trivy-action

### 除外ファイルの確認

以下のファイルが検出されないことを確認:
- [ ] `*.sops.yaml` (7ファイル)
- [ ] `gotk-components.yaml`

## Step 5: 運用開始

1. Renovateが定期的に更新をチェック
2. 新バージョン検出時にPRが自動作成
3. PRをレビューし、問題なければマージ
4. FluxCDが自動的にクラスターへ適用

## Verification Checklist

- [ ] Renovate Appがインストールされている
- [ ] `renovate.json5`がリポジトリルートに存在
- [ ] Onboarding PRがマージされている
- [ ] Dependency Dashboardが有効
- [ ] HelmReleaseが検出対象になっている
- [ ] SOPSファイルが除外されている

## Troubleshooting

### PRが作成されない場合

1. Renovate Dashboardを確認
2. ログでエラーを確認
3. `renovate.json5`の構文エラーをチェック

### 意図しないファイルが更新される場合

1. `ignorePaths`にパターンを追加
2. 設定をコミットして再実行

### レート制限に達した場合

1. `prConcurrentLimit`を下げる
2. 不要なPRをクローズ
