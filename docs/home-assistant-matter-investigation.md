# Home Assistant Matter統合 調査・作業ログ

## 概要

- **日付**: 2026-01-04/05
- **目的**: Kubernetes上のHome AssistantでMatter統合を使用し、SwitchBot Hub 2を接続する
- **結果**: **技術的に困難** - Talos Linux + Kubernetes環境ではmDNSマルチキャストが正常に動作しない

## 環境

| コンポーネント | バージョン/詳細 |
|---------------|----------------|
| Kubernetes | Talos Linux |
| Home Assistant | ghcr.io/home-assistant/home-assistant:2024.12 |
| Matter Server | ghcr.io/home-assistant-libs/python-matter-server:stable |
| デバイス | SwitchBot Hub 2 (Matter対応) |
| ネットワーク | TP-Link ルーター、192.168.0.0/24 |

## 試行した設定

### 1. 基本構成

```yaml
# Deployment設定
spec:
  hostNetwork: true  # Matter/mDNS通信に必要
  dnsPolicy: ClusterFirstWithHostNet
  nodeSelector:
    kubernetes.io/hostname: homelab-node-01  # ノード固定
```

### 2. Matter Serverサイドカー追加

```yaml
- name: matter-server
  image: ghcr.io/home-assistant-libs/python-matter-server:stable
  args:
    - --storage-path
    - /data
    - --paa-root-cert-dir
    - /data/credentials
    - --primary-interface
    - enp1s0f1
  ports:
    - containerPort: 5580
```

### 3. ネットワーク権限の追加

```yaml
securityContext:
  capabilities:
    add:
      - NET_ADMIN
      - NET_RAW
```

### 4. Privilegedモード + D-Bus

```yaml
securityContext:
  privileged: true
volumeMounts:
  - name: dbus
    mountPath: /var/run/dbus
    readOnly: true
volumes:
  - name: dbus
    hostPath:
      path: /var/run/dbus
```

### 5. Avahi環境変数

```yaml
env:
  - name: CHIP_USE_AVAHI
    value: "1"
```

## 発生したエラー

### mDNSマルチキャスト送信エラー

```
CHIP Error 0x00000046: No endpoint was available to send the message
Failed to advertise records: src/lib/dnssd/minimal_mdns/Server.cpp:344
```

### Discovery タイムアウト

```
Discovery timed out
Secure Pairing Failed
CHIP Error 0x00000003: Incorrect state
Commission with code failed for node 1.
```

## 調査結果

### ローカルマシンからのmDNS確認

```bash
# mDNSは正常に動作している
$ dns-sd -B _matterc._udp local.
8AD46F55AE0AB66B._matterc._udp.local.  # SwitchBot Hub 2が見える

# SwitchBot Hub 2のIPアドレス
$ dns-sd -G v4v6 64E833EE8F98.local.
192.168.0.206  # IPv4
FE80::66E8:33FF:FEEE:8F98  # IPv6
```

### コンテナからのネットワーク確認

```bash
# UDPパケット送信は可能
$ kubectl exec ... -- python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.sendto(b'test', ('192.168.0.206', 5540))
"
# → 成功

# ポート5353（mDNS）はリッスンしている
$ kubectl exec ... -- ss -ulnp | grep 5353
# → 多数のソケットがバインドされている
```

### Matter Server API経由でのペアリング試行

```python
# IPアドレス直接指定でコミッショニング
msg = {
    'command': 'commission_with_code',
    'args': {
        'code': '13703449605',
        'ip_addr': '192.168.0.206',
        'network_only': True
    }
}
# 結果: Discovery timed out
```

## 根本原因

1. **Matter SDK (CHIP) のmDNS実装の制限**
   - Matter SDKはmDNSマルチキャスト（224.0.0.251, ff02::fb）を使用
   - コンテナ内からマルチキャストグループへの参加/送信が正常に動作しない

2. **Talos LinuxにAvahiがない**
   - Talosはminimal OSのため、Avahiデーモンがインストールされていない
   - `CHIP_USE_AVAHI=1`を設定しても、Avahiサービスが存在しない

3. **IPアドレス直接指定でも内部でmDNS discoveryを使用**
   - Matter SDKは`ip_addr`パラメータを指定しても、内部でmDNS discoveryを必須としている

## 試行しなかった代替案

### 1. Avahiサイドカーコンテナ

Avahiデーモンを別コンテナとして起動し、D-Bus経由で連携する方法。ただし、Talosではホスト側のD-Busサポートが限定的。

### 2. mDNS Reflector/Proxy

ネットワーク上にmDNS reflectorを設置し、コンテナとLAN間のmDNSを中継する方法。

### 3. Thread Border Router

Thread対応デバイスを使用し、Thread経由でMatter通信を行う方法。

## 結論と推奨事項

### 現時点での結論

**Kubernetes + Talos Linux環境でのMatter統合は技術的に困難**

### 推奨される代替案

1. **SwitchBotクラウド統合を使用**（推奨）
   - Home Assistant → 設定 → デバイスとサービス → 統合を追加 → SwitchBot
   - APIトークンを使用してクラウド経由で制御

2. **Home Assistant OSを別マシンで実行**
   - Raspberry Piなどで直接Home Assistant OSを実行
   - Matter統合がネイティブで動作

3. **将来的な改善を待つ**
   - Matter Server / CHIP SDKのコンテナ対応改善
   - Talos LinuxへのAvahi追加（可能性は低い）

## 参考リンク

- [Home Assistant Matter Integration](https://www.home-assistant.io/integrations/matter/)
- [Python Matter Server](https://github.com/home-assistant-libs/python-matter-server)
- [SwitchBot Hub 2 Matter](https://www.switch-bot.com/products/switchbot-hub-2)
- [Talos Linux](https://www.talos.dev/)

## 関連Issue

- GitHub Issue: #74

## 変更履歴

| 日時 | 変更内容 |
|------|----------|
| 2026-01-04 | Home Assistant Deployment作成 |
| 2026-01-04 | Matter Serverサイドカー追加 |
| 2026-01-04 | 各種ネットワーク設定試行 |
| 2026-01-05 | mDNS調査、IPアドレス直接指定試行 |
| 2026-01-05 | 技術的困難と判断、ドキュメント作成 |
