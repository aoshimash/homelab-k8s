# Feature Specification: Home Assistant導入とSwitchBot Hub 2のMatter連携

**Feature Branch**: `011-home-assistant`
**Created**: 2026-01-04
**Status**: Draft
**Input**: User description: "Home Assistantを導入してSwitchBot Hub 2とMatterで連携する"
**Related Issue**: [#74](https://github.com/aoshimash/homelab-k8s/issues/74)

## Clarifications

### Session 2026-01-04

- Q: Home Assistant Podを特定のノードに固定（nodeSelector/nodeAffinity）しますか？ → A: 特定ノードに固定する（nodeSelector使用）
- Q: Home Assistant用PVCのストレージサイズはどのくらいにしますか？ → A: 5Gi
- Q: Tailscale Ingressで公開するホスト名は何にしますか？ → A: home-assistant
- Q: 初期デプロイ時にConfigMapにサンプルオートメーションを含めますか？ → A: サンプルオートメーションを含める
- Q: 使用するHome Assistantのコンテナイメージはどれにしますか？ → A: 公式イメージ（ghcr.io/home-assistant/home-assistant）

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Home AssistantへのWebアクセス (Priority: P1)

ユーザーは、Tailscale経由で外部からHome AssistantのWebインターフェースにアクセスし、スマートホームデバイスの状態確認や操作を行いたい。

**Why this priority**: Home Assistantの基本的なデプロイと外部アクセスは、すべての機能の前提条件となるため最優先。

**Independent Test**: Tailscale Ingress経由でHome AssistantのWebUIにアクセスし、ログイン・初期設定ができることを確認する。

**Acceptance Scenarios**:

1. **Given** Home AssistantがKubernetesクラスターにデプロイされている, **When** ユーザーがTailscale Ingress経由でアクセスする, **Then** Home AssistantのログインページまたはOnboarding画面が表示される
2. **Given** 初回アクセス時, **When** ユーザーがOnboarding画面で初期設定を完了する, **Then** 管理者アカウントが作成されダッシュボードが表示される
3. **Given** Home Assistantが稼働中, **When** Podが再起動する, **Then** 設定データは永続化されており、再ログイン後に以前の状態が維持されている

---

### User Story 2 - SwitchBot Hub 2とのMatter接続 (Priority: P2)

ユーザーは、SwitchBot Hub 2をMatterプロトコルでHome Assistantに接続し、ローカルネットワーク内で低遅延・高信頼性のスマートホーム制御を実現したい。

**Why this priority**: Matter連携はこの機能の主要目的であり、Home Assistantの基本デプロイ後に実現すべき機能。

**Independent Test**: SwitchBot Hub 2のMatterペアリングコードを使用してHome Assistantに登録し、デバイスが認識されることを確認する。

**Acceptance Scenarios**:

1. **Given** Home AssistantがhostNetworkモードで稼働している, **When** Matter統合を有効化する, **Then** Matterコントローラーが正常に起動する
2. **Given** SwitchBot Hub 2のMatter機能が有効化されている, **When** ペアリングコードをHome Assistantに入力する, **Then** Hub 2がデバイスとして認識・登録される
3. **Given** SwitchBot Hub 2がMatter経由で接続されている, **When** Home Assistantからデバイスを操作する, **Then** ローカルネットワーク経由で即座に（1秒以内に）操作が反映される

---

### User Story 3 - YAMLによるオートメーション管理 (Priority: P3)

ユーザーは、オートメーション設定をYAMLファイル（ConfigMap）として管理し、Gitで変更履歴を追跡したい。

**Why this priority**: オートメーションのコード管理は、デバイス接続後の運用効率化のための機能。

**Independent Test**: ConfigMapに定義したオートメーションがHome Assistantに反映され、トリガー条件で自動実行されることを確認する。

**Acceptance Scenarios**:

1. **Given** オートメーション定義を含むConfigMapがデプロイされている, **When** Home Assistantが起動する, **Then** ConfigMapのオートメーションが自動的に読み込まれる
2. **Given** ConfigMapのオートメーション定義を更新する, **When** ConfigMapを再デプロイする, **Then** Home Assistantに新しいオートメーションが反映される
3. **Given** トリガー条件を満たすイベントが発生する, **When** オートメーションが実行される, **Then** 定義されたアクションが正しく実行される

---

### Edge Cases

- Home Assistant PodとSwitchBot Hub 2が異なるL2セグメントにある場合、Matter通信は失敗する
- nodeSelectorで指定したノードが利用不可になった場合、Podは起動できずMatterデバイスも操作不可となる
- Longhorn PVCが利用不可の場合、設定データが失われる
- ConfigMapの構文エラーがある場合、オートメーションの読み込みが失敗する

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Home AssistantをKubernetesクラスターにDeploymentとしてデプロイできること
- **FR-002**: Home Assistant Podは`hostNetwork: true`で実行され、Matter通信（mDNS/IPv6リンクローカル）が可能であること
- **FR-009**: Home Assistant PodはnodeSelectorを使用して特定のノードに固定され、Matterデバイスの再ペアリングを回避すること
- **FR-003**: Home Assistant Podは`dnsPolicy: ClusterFirstWithHostNet`を使用し、Kubernetes内部DNSも利用可能であること
- **FR-004**: Longhorn PVCを使用して設定データ（/config）を永続化すること
- **FR-005**: Tailscale Ingressを経由して外部からHome AssistantのWebインターフェースにアクセスできること
- **FR-006**: Home AssistantのMatter統合を有効化し、Matterコントローラーとして機能すること
- **FR-007**: SwitchBot Hub 2をMatterプロトコルでHome Assistantに接続できること
- **FR-008**: オートメーション設定をConfigMapとして外部管理できること
- **FR-010**: 初期デプロイ時にサンプルオートメーションをConfigMapに含めること

### Key Entities

- **Home Assistant Pod**: 公式イメージ（ghcr.io/home-assistant/home-assistant）を使用したスマートホームプラットフォームコンテナ。hostNetworkモードで動作し、Matterコントローラー機能を持つ
- **Longhorn PVC**: Home Assistantの設定データ（/config）を永続化するストレージ（5Gi）
- **Tailscale Ingress**: 外部からHome Assistantへのセキュアなアクセスポイントをホスト名`home-assistant`でTailscaleネットワーク経由で提供
- **ConfigMap（オートメーション）**: YAMLで定義されたオートメーション設定を格納
- **SwitchBot Hub 2**: Matter対応のスマートホームハブデバイス

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Tailscale経由でHome AssistantのWebインターフェースに5秒以内にアクセスできる
- **SC-002**: SwitchBot Hub 2がMatter経由でHome Assistantに正常に接続され、デバイス一覧に表示される
- **SC-003**: Home AssistantからSwitchBot Hub 2配下のデバイスを操作した際、1秒以内に操作が反映される
- **SC-004**: Home Assistant Podが再起動しても、設定データとデバイス登録情報が保持される
- **SC-005**: ConfigMapに定義したオートメーションが正しく動作する

## Assumptions

- Kubernetesクラスターには既にLonghornとTailscale Operatorがデプロイされている
- Home Assistant PodとSwitchBot Hub 2は同一L2セグメント（同一物理ネットワーク）に存在する
- SwitchBot Hub 2は事前にSwitchBotアプリでMatter機能が有効化されている
- IPv6リンクローカルアドレスはホストネットワークで自動的に利用可能である
- Kubernetesノードのファイアウォールは、Matter通信に必要なポート（mDNS: UDP 5353、Matter: TCP/UDP 5540）を許可している

## Out of Scope

- Prometheusエクスポーターによるセンサーデータのメトリクス収集とGrafana可視化
- Zigbeeドングルを使用したローカルデバイス拡張
- Home Assistantの高可用性（HA）構成
- 複数ノードへのMatterデバイス分散
