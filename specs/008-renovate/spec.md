# Feature Specification: Renovate導入

**Feature Branch**: `008-renovate`  
**Created**: 2026-01-03  
**Status**: Draft  
**Input**: User description: "renovateの導入"

## Clarifications

### Session 2026-01-03

- Q: 同時オープンPR数の上限はいくつか？ → A: 10個（積極的に更新を取り込む方針）
- Q: メジャーバージョン更新の扱いは？ → A: メジャー更新は個別PRで作成（グループ化しない）

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Helm Chart更新の自動検出 (Priority: P1)

リポジトリ管理者として、HelmReleaseで管理している各コンポーネント（Cilium、Longhorn、Tailscale、Grafana Alloy、kube-state-metrics、metrics-server）のHelmチャートに新バージョンがリリースされたとき、自動的にPull Requestが作成されることで、更新の見落としを防ぎたい。

**Why this priority**: Homelabインフラの大部分がHelmRelease経由で管理されており、これらの依存関係を最新に保つことがセキュリティと安定性の観点で最も重要。

**Independent Test**: HelmRepositoryで指定されているチャートに対して、意図的にバージョンを下げた状態でRenovateを実行し、PRが自動生成されることを確認できる。

**Acceptance Scenarios**:

1. **Given** HelmReleaseのchartバージョンが最新より古い状態, **When** Renovateが定期実行される, **Then** バージョンアップデートのPRが自動作成される
2. **Given** PRが作成された状態, **When** PRの内容を確認する, **Then** 変更されたファイル、新バージョン、変更履歴（リリースノートへのリンク）が確認できる
3. **Given** 複数のHelmReleaseで更新がある状態, **When** Renovateが実行される, **Then** コンポーネントごとに個別のPRが作成される

---

### User Story 2 - GitHub Actions更新の自動検出 (Priority: P2)

リポジトリ管理者として、GitHub Actionsで使用しているアクション（actions/checkout、trivy-actionなど）に新バージョンがリリースされたとき、自動的にPull Requestが作成されることで、CIパイプラインを最新の状態に保ちたい。

**Why this priority**: CI/CDパイプラインのセキュリティと機能改善のために重要だが、クラスター本体のインフラより優先度は低い。

**Independent Test**: GitHub Actionsワークフローで使用しているアクションのバージョンを意図的に古くし、PRが生成されることを確認できる。

**Acceptance Scenarios**:

1. **Given** ワークフローで古いバージョンのアクションを使用している状態, **When** Renovateが定期実行される, **Then** アクション更新のPRが自動作成される
2. **Given** PRが作成された状態, **When** PRの内容を確認する, **Then** セマンティックバージョニングに基づいた適切な更新であることが確認できる

---

### User Story 3 - Talos/Fluxコンポーネントの更新通知 (Priority: P3)

リポジトリ管理者として、Talos LinuxやFluxCDの新バージョンがリリースされたとき、それを把握できることで、計画的なアップグレードを実施したい。

**Why this priority**: これらは手動での慎重なアップグレードが必要なコアコンポーネントであり、自動マージは適さないが、新バージョンの把握は必要。

**Independent Test**: Talos/Fluxの更新検出ルールが正しく設定され、PRまたはIssueが作成されることを確認できる。

**Acceptance Scenarios**:

1. **Given** TalosまたはFluxに新バージョンがある状態, **When** Renovateが定期実行される, **Then** 更新を通知するPRまたはDashboard Issueが作成される
2. **Given** 通知が作成された状態, **When** 内容を確認する, **Then** 手動アップグレードが必要な旨とリリースノートへのリンクが含まれる

---

### Edge Cases

- 複数の更新が同時に検出された場合、PRが大量に作成されないようレート制限が適用されるか？
- SOPS暗号化されたシークレットファイルがある場合、Renovateはそれらを正しくスキップするか？
- PRが既に存在する状態で新しいバージョンがリリースされた場合、既存PRが更新されるか新規PRが作成されるか？
- ネットワーク障害などでRenovateの実行が失敗した場合、適切にリトライされるか？

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: システムはHelmReleaseマニフェスト内のchartバージョンを検出し、新バージョンがある場合にPRを作成しなければならない
- **FR-002**: システムはGitHub Actionsワークフロー内のアクションバージョンを検出し、新バージョンがある場合にPRを作成しなければならない
- **FR-003**: システムは各依存関係ごとに個別のPRを作成しなければならない。ただしメジャーバージョン更新は必ず個別PRとし、グループ化してはならない
- **FR-004**: PRには変更内容の要約とリリースノートへのリンクが含まれなければならない
- **FR-005**: システムは定期的（少なくとも1日1回）に依存関係の更新をチェックしなければならない
- **FR-006**: システムは同時にオープンできるPR数を最大10個に制限しなければならない
- **FR-007**: システムはSOPS暗号化ファイルを更新対象から除外しなければならない
- **FR-008**: システムはFluxCDが自動生成するファイル（gotk-components.yamlなど）を更新対象から除外しなければならない

### Key Entities

- **依存関係（Dependency）**: 更新対象となるパッケージやコンポーネント（Helmチャート、GitHub Action、Dockerイメージなど）。バージョン、ソース、現在の状態を持つ
- **更新PR（Update Pull Request）**: Renovateが作成する更新提案。対象の依存関係、変更前後のバージョン、変更理由を含む
- **設定（Configuration）**: Renovateの動作を制御するルール群。スケジュール、対象パス、グループ化ルール、自動マージ設定などを含む

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: HelmReleaseの新バージョンリリースから24時間以内にPRが自動作成される
- **SC-002**: GitHub Actionsの新バージョンリリースから24時間以内にPRが自動作成される
- **SC-003**: 作成されるPRの100%にリリースノートへのリンクが含まれる
- **SC-004**: SOPS暗号化ファイルやFlux自動生成ファイルに対する誤った更新PRが0件である
- **SC-005**: 依存関係の更新漏れが手動確認で発見される頻度が月1回未満に減少する

## Assumptions

- GitHubのRenovate Appを使用する（自己ホスト型ではなく）
- renovate.jsonまたはrenovate.json5をリポジトリルートに配置して設定を管理する
- HelmRepositoryで定義されているHelm Chartレジストリは公開されており、Renovateからアクセス可能
- 自動マージは行わず、すべてのPRは手動レビュー後にマージする運用とする
