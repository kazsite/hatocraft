# Re:ゼロから始めるMinecraft Serverの建て方 in AWS

## はじめに
このドキュメントでは、**AWS EC2**を利用してMinecraft Serverを構築する方法をまとめます。

分かり易さ重視で、最小構成の構築になります。下記指示通りに構築した際、セキュリティリスクがあると思いますが、その点は各々調整お願いします。

また、セキュリティ問題による損失の責任は一切負いませんのでご了承ください。

---

## 前提条件
- AWSアカウントを作成済み（未作成の方は[AWS アカウント作成の流れ](https://aws.amazon.com/jp/register-flow/)参照）
- 東京リージョンで構築
- Minecraft Java Editionを使用

---

## 1. VPCの構築
### 1.1 VPC作成
1. **AWS マネジメントコンソール**にログイン
2. **VPC → VPC ダッシュボード → VPCを作成**を選択
   - 作成するリソース: VPCのみ
   - 名前タグ例: `minecraft-vpc`
   - IPv4 CIDR ブロック: IPv4 CIDR の手動入力
   - IPv4 CIDR: `172.16.0.0/16`
   - IPv6 CIDR ブロック: IPv6 CIDR ブロックなし
   - テナンシー: デフォルト
3. VPC作成後、**アクション → VPC の設定を編集 → DNS 設定 → DNS ホスト名を有効化**をオン

### 1.2 サブネット作成
1. VPC に紐づけて**サブネット**を作成
   - VPC ID: 上記VPC選択
   - サブネット名例: `minecraft-subnet-public`
   - アベイラビリティーゾーン: 任意のAZを選択
   - IPv4 サブネット CIDR ブロック: `172.16.0.0/24`

### 1.3 インターネットゲートウェイ (IGW)を作成
1. **インターネットゲートウェイの作成**を選択
    - 名前タグ例: `minecraft-igw`
2. 作成した VPC にアタッチ

### 1.4 ルートテーブル設定
1. **ルートテーブルを作成**を選択
   - 名前タグ例: `minecraft-rtb-public`
   - VPC: 上記VPC選択
2. **サブネットの関連付け → 明示的なサブネットの関連付けを編集**を選択し`minecraft-subnet-public`を関連付ける
3. **ルート → ルートを編集 → ルートを追加**を選択
    - 送信先: `0.0.0.0/0`
    - ターゲット: インターネットゲートウェイ（上記IGWを選択）

---

## 2. EC2の構築
### 2.1 セキュリティグループの作成
1. **AWS マネジメントコンソール**に戻り**EC2**を選択
2. **ネットワーク & セキュリティ → セキュリティグループ → セキュリティグループを作成**を選択
   - セキュリティグループ名例: `minecraft-sg`
   - 説明: 適当に...
   - VPC: `minecraft-vpc`
3. インバウンドルールを追加
   - SSH: TCP 22 (ソース: **自　分　の　I　P**)
   - Minecraft: TCP 25565 (ソース: 0.0.0.0/0)

### 2.2 キーペアを作成
1. キーペアを作成
   - キーペア名例: `minecraft-key`
   - キーペアタイプ: RSA
   - プライベートキーファイル形式: pem

### 2.3 EC2インスタンスの作成
1. **インスタンス → インスタンスを起動**を選択
   - 名前例: `minecraft-server-ec2`
   - AMI: Ubuntu 24.04 LTS
   - アーキテクチャ: 互換性重視`x86`　コスト重視`Arm`
   - インスタンスタイプ: `m6g.medium` (どれを選ぶかはケースバイケース)
   - キーペア名: `minecraft-key`
2. ネットワーク設定
   - VPC: `minecraft-vpc`
   - サブネット: `minecraft-subnet-public`
   - パブリック IP の自動割り当て: 有効化
   - セキュリティグループ: `minecraft-sg`
3. ストレージの設定
   - サイズ: 16 GiB
   - ボリュームタイプ: gp3
   - IOPS: 3000
   - 終了時に削除: いいえ
   - 暗号化: 暗号化済み
   - KMS キー: (デフォルト)aws/ebs
   - スループット: 125

---

## 3. 必要ソフトウェアのインストール
### 3.1 SSH接続
```bash
ssh -i /path/to/minecraft-key.pem ubuntu@ec2-xxx-xxx-xxx-xxx.ap-northeast-1.compute.amazonaws.com
```
### 3.2 Ubuntu初期設定
```bash
# パッケージ更新
sudo apt update && sudo apt upgrade -y

# NTPクライント & Docker導入
git clone https://github.com/kazsite/hatocraft.git
chmod +x ./hatocraft/ec2-ubuntu-setup.sh
sudo ./hatocraft/ec2-ubuntu-setup.sh
```

### 3.3 Minecraft Server導入
SSH再ログイン後以下コマンド実行
```bash
# docker-compose.yml実行
cd hatocraft
docker compose up -d
```
MODサーバーや、server.properties設定等詳しい方法は[Minecraft Server on Docker](https://docker-minecraft-server.readthedocs.io/en/latest/)参照

### 3.4 マルチプレイログイン
1. Minecraft Java Edtion実行後、**マルチプレイ → サーバーを追加 → サーバーアドレス**にEC2パブリックIPを指定
2. <span style="color:pink; font-weight:bold;">ログイン !!!</span>
