# システム運用詳細設計書

---

## 文書情報

| 項目 | 内容 |
|------|------|
| システム名 | Dify ワークショップ環境 |
| 版 | 1.0 |
| 作成会社 | 鈴与システムテクノロジー株式会社 |

## 作成・承認履歴

| 役割 | 氏名 | 日付 |
|------|------|------|
| 弊社作成者 | 渡瀬陽己 | 2025-01-30 |
| 弊社確認者 | - | - |
| 弊社承認者 | - | - |

---

## 改訂履歴

| 版 | 改訂内容・改訂経緯 | 作成日 | 作成者 |
|---|---|---|---|
| 1.0 | 新規作成 | 2025-01-30 | 渡瀬 |

## 目次
- [1. 詳細アーキテクチャ設計](#1-詳細アーキテクチャ設計)
- [2. Dify環境構築手順詳細](#2-dify環境構築手順詳細)
- [3. ネットワーク詳細設計](#3-ネットワーク詳細設計)
- [4. セキュリティ詳細設計](#4-セキュリティ詳細設計)
- [5. 監視・ログ詳細設計](#5-監視ログ詳細設計)
- [6. 運用手順書](#6-運用手順書)
- [7. 障害対応手順](#7-障害対応手順)

## 1. 詳細アーキテクチャ設計

### 1-1. システム構成詳細

#### EC2インスタンス仕様
| 項目 | 仕様 |
|------|------|
| インスタンスタイプ | t3.medium |
| OS | Amazon Linux 2023 |
| CPU | 2vCPU |
| メモリ | 4GB |
| ストレージ | 30GB gp3 |
| サブネット | プライベート(既存環境) |
| セキュリティグループ | dify-workshop-sg |

#### Difyアプリケーション構成

| サービス名 | 技術スタック | ポート | 用途 | 備考 |
|------------|-------------|------|------|------|
| dify-web | React.js | 3000 | フロントエンドUI | Dify公式が提供しているものを使用 |
| dify-api | Python Flask | 5001 | バックエンドAPI | Dify公式が提供しているものを使用 |
| dify-worker | Python | - | バックグラウンド処理 | Dify公式が提供しているものを使用 |
| redis | Redis | 6379 | セッション・キャッシュ | Dify公式が提供しているものを使用 |
| postgresql | PostgreSQL | 5432 | メインデータベース | Dify公式が提供しているものを使用 |
| nginx | Nginx | 80 | リバースプロキシ | **SSL終端・ALB連携対応のため設定変更** |
| ssrf_proxy | Squid | 3128 | アウトバウンド通信制御 | **Bedrock・SMTP通信制御のため設定変更** |

> 上記のうち、nginxとssrf_proxyのみSSL終端やアウトバウンド通信要件に合わせて設定・設計を変更し、それ以外はDify公式が提供しているものを使用する。

### 1-2. ネットワーク構成詳細

#### 既存ネットワーク環境利用

| ネットワーク要素 | CIDR/設定値 | 配置リソース | 管理部署 |
|------------------|-------------|--------------|----------|
| 既存VPC | 10.x.x.x/16 | 全リソース | ネットワークサポート部 |
| パブリックサブネット | 10.x.1.x/24 (ap-northeast-1a,1c) | ALB, NAT Gateway | ネットワークサポート部 |
| プライベートサブネット | 10.x.2.x/24 (ap-northeast-1a) | EC2インスタンス | ネットワークサポート部 |
| Internet Gateway | 既存環境利用 | インターネット接続 | ネットワークサポート部 |
| NAT Gateway | 既存環境利用 | プライベート外部通信 | ネットワークサポート部 |
| Virtual Private Gateway | 既存環境利用 | 社内接続 | ネットワークサポート部 |
| Direct Connect | 既存環境利用 | 専用線接続 | ネットワークサポート部 |

#### ALB設定詳細
| 項目 | 設定値 |
|------|--------|
| スキーム | internet-facing |
| IPアドレスタイプ | ipv4 |
| リスナー | HTTPS:443 |
| SSL証明書 | ACM管理証明書 |
| ターゲットグループ | dify-workshop-tg |
| ヘルスチェック | HTTP:80/health |

### 1-3. Amazon Bedrock連携設計

#### Bedrock設定
| 項目 | 設定値 |
|------|--------|
| モデル | Claude 3.7 Sonnet |
| リージョン | ap-northeast-1 |
| アクセス方式 | IAMロール |
| API呼び出し | HTTPS/443 |
| レート制限 | 1000 requests/minute |

## 2. CloudFormationによるインフラ構築設計

### 2-1. CloudFormationで構築するリソース概要

当システムでは、Infrastructure as Code（IaC）の原則に従い、CloudFormationテンプレートを使用してAWSリソースを構築する。既存のネットワーク環境（VPC、サブネット、Direct Connect等）はネットワークサポート部が管理する共有リソースを利用し、Difyワークショップ専用のリソースのみを新規作成する。

#### 構築対象リソース一覧
| リソース種別 | リソース名 | 用途 | 備考 |
|--------------|------------|------|------|
| IAMロール | DifyWorkshopRole | EC2インスタンス用権限 | Bedrock・CloudWatch・SSM権限 |
| インスタンスプロファイル | DifyWorkshopInstanceProfile | IAMロール関連付け | EC2にロール適用 |
| セキュリティグループ | DifyWorkshopSecurityGroup | EC2用通信制御 | ALBからのHTTP許可 |
| セキュリティグループ | ALBSecurityGroup | ALB用通信制御 | インターネットからのHTTPS許可 |
| EC2インスタンス | DifyWorkshopInstance | Difyアプリケーション実行環境 | t3.medium、プライベートサブネット |
| ターゲットグループ | DifyTargetGroup | ALB負荷分散先 | ヘルスチェック設定 |
| Application Load Balancer | DifyALB | 外部アクセス受付 | HTTPS/HTTP リスナー |
| SSL証明書 | DifySSLCertificate | HTTPS通信用 | ACM管理、DNS検証 |

### 2-2. IAMロール・ポリシー詳細設計

#### DifyWorkshopRole 設計
EC2インスタンスがAWSサービスにアクセスするためのIAMロールを作成する。最小権限の原則に従い、必要な権限のみを付与する。

**信頼関係ポリシー**
- プリンシパル: ec2.amazonaws.com
- アクション: sts:AssumeRole
- 用途: EC2インスタンスがロールを引き受け可能にする

**アタッチするAWS管理ポリシー**
| ポリシー名 | 用途 | 権限内容 |
|------------|------|----------|
| CloudWatchAgentServerPolicy | CloudWatch監視 | メトリクス・ログ送信権限 |
| AmazonSSMManagedInstanceCore | Systems Manager | Session Manager接続権限 |

**カスタムポリシー詳細**

1. **BedrockAccessPolicy**
   - 目的: Amazon Bedrock Claude 3.7 Sonnetモデルへのアクセス
   - 許可アクション:
     - bedrock:InvokeModel
     - bedrock:InvokeModelWithResponseStream
   - リソース制限: anthropic.claude-3-sonnet-20240229-v1:0 のみ
   - セキュリティ考慮: 特定モデルのみアクセス可能

2. **CloudWatchLogsPolicy**
   - 目的: Difyアプリケーションログの CloudWatch Logs 送信
   - 許可アクション:
     - logs:CreateLogGroup
     - logs:CreateLogStream
     - logs:PutLogEvents
     - logs:DescribeLogStreams
   - リソース制限: /aws/dify/* ロググループのみ
   - セキュリティ考慮: Dify関連ログのみアクセス可能

### 2-3. EC2インスタンス詳細設計

#### インスタンス仕様
| 項目 | 設定値 | 選定理由 |
|------|--------|----------|
| AMI | ami-0d52744d6551d851e | Amazon Linux 2023最新版 |
| インスタンスタイプ | t3.medium | CPU 2vCPU、メモリ 4GB、ワークショップに適切 |
| サブネット | 既存プライベートサブネット | セキュリティ確保、既存環境利用 |
| セキュリティグループ | DifyWorkshopSecurityGroup | 専用SG、最小権限通信 |
| IAMインスタンスプロファイル | DifyWorkshopInstanceProfile | 必要最小権限のロール |
| キーペア | 指定なし | Session Manager使用のため不要 |

#### ストレージ設計
| デバイス | サイズ | タイプ | 暗号化 | 用途 |
|----------|--------|--------|--------|------|
| /dev/xvda | 30GB | gp3 | 有効 | OS・アプリケーション領域 |

#### 構築手順（UserData非使用・手動構築）

1. **EC2インスタンス起動後、Session Managerでログインする。**
2. **以下のコマンドを順に手動実行し、Dify環境を構築する。**

   ```sh
   # システムパッケージ更新
   sudo yum update -y

   # Docker・Gitインストール
   sudo amazon-linux-extras install docker -y
   sudo yum install git -y

   # Dockerサービス起動・自動起動設定
   sudo systemctl start docker
   sudo systemctl enable docker

   # ec2-userをdockerグループに追加
   sudo usermod -aG docker ec2-user

   # Docker Composeインストール
   sudo curl -L "https://github.com/docker/compose/releases/download/v2.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose

   # CloudWatch Agentインストール
   sudo yum install amazon-cloudwatch-agent -y

   # アプリケーションディレクトリ作成・権限設定
   sudo mkdir -p /opt/dify
   sudo chown ec2-user:ec2-user /opt/dify
   ```

3. **以降は「Difyアプリケーション構築フェーズ」に従い、ソース取得・環境変数設定・docker compose起動等を手動で実施する。**

### 2-4. ネットワークリソース詳細設計

#### セキュリティグループ設計

**DifyWorkshopSecurityGroup（EC2用）**
- 目的: EC2インスタンスの通信制御
- インバウンドルール:
  - TCP/80: ALBSecurityGroupからのHTTP通信のみ許可
- アウトバウンドルール:
  - TCP/443: 全てのIPアドレスへのHTTPS通信（Bedrock API用）
  - TCP/25: 社内SMTPサーバー（10.114.2.8）へのSMTP通信
  - TCP/80: 全てのIPアドレスへのHTTP通信（パッケージ更新用）

**ALBSecurityGroup（ALB用）**
- 目的: Application Load Balancerの通信制御
- インバウンドルール:
  - TCP/443: 全てのIPアドレスからのHTTPS通信
  - TCP/80: 全てのIPアドレスからのHTTP通信（HTTPS リダイレクト用）
- アウトバウンドルール: デフォルト（全許可）

#### Application Load Balancer設計

**基本設定**
| 項目 | 設定値 | 理由 |
|------|--------|------|
| スキーム | internet-facing | インターネットからのアクセス受付 |
| IPアドレスタイプ | ipv4 | IPv4のみ対応 |
| サブネット | 既存パブリックサブネット（2AZ） | 高可用性確保 |
| セキュリティグループ | ALBSecurityGroup | 専用SG適用 |

**リスナー設定**
1. **HTTPSリスナー（ポート443）**
   - SSL証明書: ACM管理証明書使用
   - デフォルトアクション: DifyTargetGroupへ転送
   - セキュリティポリシー: ELBSecurityPolicy-TLS-1-2-2017-01

2. **HTTPリスナー（ポート80）**
   - デフォルトアクション: HTTPS（ポート443）へリダイレクト
   - ステータスコード: HTTP_301（恒久的リダイレクト）

#### ターゲットグループ設計

**DifyTargetGroup設定**
| 項目 | 設定値 | 理由 |
|------|--------|------|
| プロトコル | HTTP | ALB-EC2間は内部通信 |
| ポート | 80 | Nginxリバースプロキシポート |
| ターゲットタイプ | instance | EC2インスタンス直接指定 |
| VPC | 既存VPC | 既存環境利用 |

**ヘルスチェック設定**
| 項目 | 設定値 | 理由 |
|------|--------|------|
| パス | /health | Difyアプリケーション専用エンドポイント |
| 間隔 | 30秒 | 適切な監視頻度 |
| タイムアウト | 5秒 | レスポンス待機時間 |
| 正常閾値 | 2回 | 復旧判定回数 |
| 異常閾値 | 3回 | 障害判定回数 |

#### SSL証明書設計

**DifySSLCertificate設定**
| 項目 | 設定値 | 理由 |
|------|--------|------|
| ドメイン名 | dify-workshop-{Environment}.sst-web.com | 環境別ドメイン |
| 検証方法 | DNS検証 | 自動検証、管理容易 |
| 証明書タイプ | RSA-2048 | 標準的なセキュリティレベル |
| 自動更新 | 有効 | ACMによる自動更新 |

### 2-5. CloudFormationパラメータ設計

#### 必須パラメータ
| パラメータ名 | タイプ | デフォルト値 | 説明 |
|--------------|--------|--------------|------|
| Environment | String | workshop | 環境識別子（リソース名に使用） |
| ProjectName | String | AI-Education | プロジェクト名（タグ付けに使用） |
| ExistingVPCId | AWS::EC2::VPC::Id | - | 既存VPC ID（ネットワークサポート部管理） |
| ExistingPrivateSubnetId | AWS::EC2::Subnet::Id | - | EC2配置用既存プライベートサブネット ID |
| ExistingPublicSubnetIds | List<AWS::EC2::Subnet::Id> | - | ALB配置用既存パブリックサブネット ID（2AZ） |

#### 出力値設計
| 出力キー | 説明 | エクスポート名 | 用途 |
|----------|------|----------------|------|
| EC2InstanceId | 作成されたEC2インスタンスID | {StackName}-EC2InstanceId | 運用・管理作業 |
| ALBDNSName | ALBのDNS名 | {StackName}-ALBDNSName | アクセスURL確認 |
| TargetGroupArn | ターゲットグループARN | {StackName}-TargetGroupArn | 監視設定 |
| SSLCertificateArn | SSL証明書ARN | {StackName}-SSLCertificateArn | 証明書管理 |

### 2-6. Dify環境構築手順

#### Step 1: CloudFormationスタック作成
CloudFormationテンプレートを使用してAWSリソースを一括作成する。既存のネットワーク環境情報をパラメータとして指定し、Difyワークショップ専用リソースを構築する。

#### Step 2: スタック作成状態確認
CloudFormationスタックの作成完了を確認し、UserDataスクリプトによる基本環境セットアップの完了を待機する。EC2インスタンスの起動とDocker環境の準備が自動的に実行される。

#### Step 3: リソース情報取得
CloudFormationの出力値から作成されたリソースの情報を取得し、後続の作業で使用する。EC2インスタンスID、ALB DNS名等の重要な情報を確認する。

### 2-7. Difyアプリケーション構築フェーズ

#### Step 1: EC2インスタンスへ接続・Difyソースコード取得
CloudFormationで作成されたEC2インスタンスにSession Managerで接続し、Difyアプリケーションのソースコードを取得する。UserDataスクリプトにより、既にDocker環境とアプリケーションディレクトリが準備されている。

#### Step 2: 環境設定ファイル作成

**環境変数設定項目一覧**

**Dify基本設定**
- CONSOLE_WEB_URL: https://dify-workshop.example.com
- CONSOLE_API_URL: https://dify-workshop.example.com
- SERVICE_API_URL: https://dify-workshop.example.com

**データベース設定**
- DB_USERNAME: dify
- DB_PASSWORD: dify_password_2025
- DB_HOST: db
- DB_PORT: 5432
- DB_DATABASE: dify

**Redis設定**
- REDIS_HOST: redis
- REDIS_PORT: 6379
- REDIS_PASSWORD: redis_password_2025

**AWS Bedrock設定**
- AWS_REGION: ap-northeast-1
- BEDROCK_MODEL: anthropic.claude-3-sonnet-20240229-v1:0
- HTTP_PROXY: http://ssrf_proxy:3128
- HTTPS_PROXY: http://ssrf_proxy:3128

**セキュリティ設定**
- SECRET_KEY: workshop_secret_key_2025_dify
- ENCRYPT_PUBLIC_KEY: workshop_encrypt_key_2025

**メール設定（社内SMTP使用）**
- MAIL_TYPE: smtp
- MAIL_DEFAULT_SEND_FROM: dify-workshop@sst-web.com
- SMTP_SERVER: 10.114.2.8
- SMTP_PORT: 25
- SMTP_USE_TLS: false
- SMTP_PROXY: http://ssrf_proxy:3128

**機能フラグ**
- ENABLE_SIGNUP: true
- ENABLE_EMAIL_CODE_LOGIN: true
- ENABLE_EMAIL_PASSWORD_LOGIN: true
- ENABLE_SOCIAL_OAUTH_LOGIN: false

**ワークショップ専用設定**
- MAX_ACTIVE_REQUESTS: 100
- UPLOAD_FILE_SIZE_LIMIT: 15
- UPLOAD_FILE_BATCH_LIMIT: 5

#### Step 3: Docker Compose設定カスタマイズ

**Docker Composeサービス構成**

- Dify公式が提供しているdocker-compose.ymlをベースとし、**nginx**と**ssrf_proxy**サービスのみ以下の通りカスタマイズする。

**Nginxサービス (nginx) の主な変更点**
- ALB連携・SSL終端対応のため、リバースプロキシ設定をALBのX-Forwarded-*ヘッダー信頼・ヘルスチェック対応に変更
- サーバー名・最大ボディサイズ・アップストリーム設定等をワークショップ要件に合わせて調整

**SSRF Proxyサービス (ssrf_proxy) の主な変更点**
- Bedrock・SMTP通信のみ許可するようsquid.confをカスタマイズ
- 必要なアウトバウンド先（Bedrock, 社内SMTP）以外はdeny設定
- DX/VGW経由ルーティングに対応

#### Step 4: Nginx設定

**Nginx基本設定**
- リッスンポート: 80
- サーバー名: _（全てのホスト名）
- 最大ボディサイズ: 15MB
- プロキシ信頼設定: ALBのIPレンジを信頼、X-Forwarded-*ヘッダー処理

**アップストリーム設定**
- api: server api:5001
- web: server web:3000

**ロケーション設定**

**/health（ヘルスチェック）**
- アクセスログ: off
- レスポンス: 200 "healthy"
- Content-Type: text/plain

**/v1（API エンドポイント）**
- プロキシ先: http://api
- タイムアウト: 読み取り300秒、送信300秒
- ヘッダー: Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto

**/console/api（コンソールAPI）**
- プロキシ先: http://api
- タイムアウト: 読み取り300秒、送信300秒
- ヘッダー: Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto

**/（Webフロントエンド）**
- プロキシ先: http://web
- ヘッダー: Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto

## 2-3. 起動・初期設定フェーズ

#### Step 1: アプリケーション起動

**起動コマンド**
- Docker Compose起動: `docker-compose -f docker-compose.workshop.yml up -d`
- 起動確認: `docker-compose -f docker-compose.workshop.yml ps`
- ログ確認: `docker-compose -f docker-compose.workshop.yml logs -f`

#### Step 2: データベース初期化

**初期化コマンド**
- マイグレーション実行: `docker-compose -f docker-compose.workshop.yml exec api flask db upgrade`
- 管理者アカウント作成: `docker-compose -f docker-compose.workshop.yml exec api flask create-admin-account --email admin@sst-web.com --password AdminPassword2025`

#### Step 3: アプリケーション起動確認
DifyアプリケーションのDocker Compose構成を起動し、各コンテナの正常動作を確認する。データベースの初期化、管理者アカウントの作成、ALBヘルスチェックの正常化を実施する。

## 3. ネットワーク詳細設計

### 3-1. 通信要件詳細一覧

#### 外部通信要件
| 通信種別 | 接続元 | 接続先 | プロトコル/ポート | 経由 | 用途 | 備考 |
|----------|--------|--------|-------------------|------|------|------|
| HTTPS通信 | 学生・講師 | ALB | TCP/443 | IGW | Dify Webアクセス | 既存ALB使用 |
| HTTPS通信 | 学生・講師 | ALB | TCP/443 | IGW | Dify APIアクセス | 既存ALB使用 |
| 管理者アクセス | 運用者 | ALB | TCP/443 | IGW | 管理画面アクセス | 管理者権限 |

#### 内部通信要件
| 通信種別 | 接続元 | 接続先 | プロトコル/ポート | 経由 | 用途 | 備考 |
|----------|--------|--------|-------------------|------|------|------|
| HTTP通信 | ALB | EC2(10.x.2.xxx) | TCP/80 | VPC内 | リバースプロキシ | Nginx経由 |
| API通信 | Nginx | dify-api | TCP/5001 | Docker内 | バックエンドAPI | コンテナ間通信 |
| Web通信 | Nginx | dify-web | TCP/3000 | Docker内 | フロントエンド | コンテナ間通信 |
| DB通信 | dify-api | PostgreSQL | TCP/5432 | Docker内 | データベース接続 | コンテナ間通信 |
| Cache通信 | dify-api | Redis | TCP/6379 | Docker内 | キャッシュ・セッション | コンテナ間通信 |
| AI通信 | EC2 | Bedrock | HTTPS/443 | ssrf_proxy → NAT Gateway | AI機能呼び出し | プロキシ経由で制御 |
| メール通信 | EC2 | 社内SMTP(10.114.2.8) | SMTP/25 | ssrf_proxy → VGW + Direct Connect | 通知メール送信 | プロキシ経由で制御 |
| 管理通信 | 運用者 | EC2 | Session Manager | AWS Systems Manager | サーバー管理 | IAMロール必須 |

### 3-2. セキュリティグループ詳細

#### Difyワークショップ用セキュリティグループ

**基本情報**
- グループ名: dify-workshop-sg
- 説明: Security group for Dify workshop environment

**セキュリティグループルール**

**インバウンドルール**
- TCP/80: ALBからのHTTPアクセス（sg-existing-alb）

**アウトバウンドルール**
- TCP/3128: ssrf_proxyコンテナへのプロキシ通信
- TCP/443: Bedrock向けHTTPS通信（ssrf_proxy経由）
- TCP/25: 社内メールサーバー向けSMTP（ssrf_proxy経由）
- TCP/80: パッケージ更新用HTTP通信

### 3-3. 既存ネットワーク環境利用

#### 利用する既存リソース
| リソース種別 | リソース名 | 用途 | 管理部署 |
|--------------|------------|------|----------|
| VPC | vpc-existing | ネットワーク基盤 | ネットワークサポート部 |
| パブリックサブネット | subnet-existing-public-1a,1c | ALB配置 | ネットワークサポート部 |
| プライベートサブネット | subnet-existing-private-1a | EC2配置 | ネットワークサポート部 |
| Internet Gateway | igw-existing | インターネット接続 | ネットワークサポート部 |
| NAT Gateway | nat-existing | プライベート外部通信 | ネットワークサポート部 |
| Virtual Private Gateway | vgw-existing | 社内接続 | ネットワークサポート部 |
| Direct Connect | dx-existing | 専用線接続 | ネットワークサポート部 |

## 4. セキュリティ詳細設計

### 4-1. IAMロール・ポリシー詳細

#### EC2インスタンス用IAMロール

**IAMポリシー設定**

**Bedrockアクセス権限**
- 許可アクション: bedrock:InvokeModel, bedrock:InvokeModelWithResponseStream
- リソース: arn:aws:bedrock:ap-northeast-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0

**CloudWatch Logs権限**
- 許可アクション: logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents, logs:DescribeLogStreams
- リソース: arn:aws:logs:ap-northeast-1:*:log-group:/aws/dify/*

**CloudWatchメトリクス権限**
- 許可アクション: cloudwatch:PutMetricData
- リソース: *（全リソース）

#### Difyアプリケーション用環境変数

**AWS認証設定**
- AWS_REGION: ap-northeast-1
- AWS_DEFAULT_REGION: ap-northeast-1
- BEDROCK_ACCESS_KEY_ID: 空文字列（IAMロール使用）
- BEDROCK_SECRET_ACCESS_KEY: 空文字列（IAMロール使用）
- BEDROCK_REGION: ap-northeast-1

### 4-2. アプリケーションレベルセキュリティ

#### Difyセキュリティ設定

**認証設定**
- ENABLE_SIGNUP: True
- ENABLE_EMAIL_CODE_LOGIN: True
- ENABLE_EMAIL_PASSWORD_LOGIN: True
- ENABLE_SOCIAL_OAUTH_LOGIN: False

**セッション設定**
- SESSION_TIMEOUT: 3600秒（1時間）
- SESSION_COOKIE_SECURE: True
- SESSION_COOKIE_HTTPONLY: True

**ファイルアップロード制限**
- UPLOAD_FILE_SIZE_LIMIT: 15MB
- UPLOAD_FILE_BATCH_LIMIT: 5ファイル
- ALLOWED_FILE_EXTENSIONS: .txt, .pdf, .docx, .md

**API制限**
- API_RATE_LIMIT: 100リクエスト/分
- MAX_ACTIVE_REQUESTS: 100同時リクエスト

**セキュリティヘッダー**
- X-Content-Type-Options: nosniff
- X-Frame-Options: DENY
- X-XSS-Protection: 1; mode=block
- Strict-Transport-Security: max-age=31536000; includeSubDomains

### 4-3. データ暗号化設定

#### 保存時暗号化
- **EBSボリューム**: AWS KMS (aws/ebs) による暗号化
- **PostgreSQLデータ**: アプリケーションレベル暗号化
- **Redis**: パスワード認証 + TLS接続

#### 通信暗号化
- **外部通信**: TLS 1.2以上
- **内部通信**: Docker内部ネットワーク（隔離済み）

## 5. 監視・ログ詳細設計

### 5-1. CloudWatch監視設定

#### カスタムメトリクス設定

**CloudWatch Agent設定パラメータ**

**メトリクス設定**
- ネームスペース: Dify/Workshop
- 収集間隔: 60秒

**CPUメトリクス**
- 測定項目: cpu_usage_idle, cpu_usage_iowait, cpu_usage_user, cpu_usage_system
- 収集間隔: 60秒

**ディスクメトリクス**
- 測定項目: used_percent
- 収集間隔: 60秒
- リソース: *（全ディスク）

**メモリメトリクス**
- 測定項目: mem_used_percent
- 収集間隔: 60秒

**ネットワークメトリクス**
- 測定項目: tcp_established, tcp_time_wait
- 収集間隔: 60秒

**ログ収集設定**

**APIログ**
- ファイルパス: /opt/dify/logs/api.log
- ロググループ: /aws/dify/api
- ログストリーム: {instance_id}

**Workerログ**
- ファイルパス: /opt/dify/logs/worker.log
- ロググループ: /aws/dify/worker
- ログストリーム: {instance_id}

**Nginxログ**
- ファイルパス: /opt/dify/logs/nginx.log
- ロググループ: /aws/dify/nginx
- ログストリーム: {instance_id}

#### Dockerコンテナ監視

**Docker監視スクリプト設定**

**基本設定**
- スクリプトパス: /usr/local/bin/docker-monitor.sh
- ネームスペース: Dify/Workshop
- 監視間隔: 60秒

**収集メトリクス**
- CPU使用率: ContainerCPUUtilization
- メモリ使用率: ContainerMemoryUtilization
- 単位: Percent

**ディメンション**
- InstanceId: EC2インスタンスID
- Container: コンテナ名

**データ収集方法**
- docker statsコマンド使用
- CloudWatch put-metric-data APIで送信
- メタデータサービスからインスタンスID取得

### 5-2. アラート設定

#### 重要度別アラート設定
| 重要度 | 項目 | 閾値 | 通知先 | アクション |
|--------|------|------|--------|------------|
| Critical | EC2 CPU使用率 | 90% | 運用者メール | 即座対応 |
| Critical | メモリ使用率 | 90% | 運用者メール | 即座対応 |
| Critical | ディスク使用率 | 85% | 運用者メール | 即座対応 |
| Warning | ALBレスポンス時間 | 5秒 | 運用者メール | 監視強化 |
| Warning | ALBエラー率 | 5% | 運用者メール | 監視強化 |
| Info | Difyアプリケーション起動 | - | 運用者メール | 記録のみ |

#### CloudWatchアラーム設定例

**CPU使用率アラーム**
- アラーム名: Dify-Workshop-High-CPU
- 説明: Dify Workshop EC2 High CPU Usage
- メトリクス名: CPUUtilization
- ネームスペース: AWS/EC2
- 統計: Average
- 期間: 300秒
- 闾値: 90%
- 比較演算子: GreaterThanThreshold
- 評価期間: 2回
- アラームアクション: arn:aws:sns:ap-northeast-1:account:dify-alerts
- ディメンション: InstanceId=i-xxxxxxxxx

**メモリ使用率アラーム**
- アラーム名: Dify-Workshop-High-Memory
- 説明: Dify Workshop High Memory Usage
- メトリクス名: MemoryUtilization
- ネームスペース: Dify/Workshop
- 統計: Average
- 期間: 300秒
- 闾値: 90%
- 比較演算子: GreaterThanThreshold
- 評価期間: 2回
- アラームアクション: arn:aws:sns:ap-northeast-1:account:dify-alerts

## 6. 運用手順書

### 6-1. 日次運用手順

#### ワークショップ開始前チェックリスト

**日次チェックスクリプト項目**

**基本情報**
- スクリプト名: Dify Workshop Daily Check Script
- 実行タイミング: ワークショップ開始30分前

**チェック項目一覧**

**1. EC2インスタンス状態確認**
- コマンド: `aws ec2 describe-instances --instance-ids i-xxxxxxxxx --query 'Reservations[0].Instances[0].State.Name'`
- 確認内容: インスタンスの稼働状態

**2. Dockerコンテナ状態確認**
- コマンド: `docker-compose -f /opt/dify/docker-compose.workshop.yml ps`
- 確認内容: 全コンテナの起動状態

**3. アプリケーション疎通確認**
- コマンド: `curl -s -o /dev/null -w "%{http_code}" http://localhost/health`
- 確認内容: ヘルスチェックエンドポイントの応答

**4. データベース接続確認**
- コマンド: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec -T db pg_isready -U dify`
- 確認内容: PostgreSQLデータベースの接続状態

**5. Redis接続確認**
- コマンド: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec -T redis redis-cli -a redis_password_2025 ping`
- 確認内容: Redisキャッシュサーバーの応答

**6. ディスク使用量確認**
- コマンド: `df -h /`
- 確認内容: ルートファイルシステムの使用率

**7. メモリ使用量確認**
- コマンド: `free -h`
- 確認内容: システムメモリの使用状態

**8. ログエラー確認**
- コマンド: `docker-compose -f /opt/dify/docker-compose.workshop.yml logs --tail=50 | grep -i error`
- 確認内容: 直近のエラーログの有無

### 6-2. ワークショップ運用手順

#### ワークショップ開始手順
1. **事前準備** (開始30分前)

**事前準備チェック項目**
- CloudFormationスタック状態確認: `aws cloudformation describe-stacks --stack-name dify-workshop-stack --query 'Stacks[0].StackStatus'`
- システム状態確認: `/usr/local/bin/daily-check.sh`
- 参加者アカウント準備確認: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec api flask list-users`
- Bedrock接続確認: `aws bedrock list-foundation-models --region ap-northeast-1`

2. **ワークショップ開始** (開始時)
   - 参加者への接続URL通知
   - 初期ログイン支援
   - 基本操作説明

3. **進行中監視** (実施中)

**監視コマンド**
- リアルタイム監視: `watch -n 30 'docker stats --no-stream'`
- アクセスログ監視: `tail -f /opt/dify/logs/nginx.log`

#### ワークショップ終了手順
1. **データバックアップ** (終了後)

**バックアップコマンド**
- データベースバックアップ: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec db pg_dump -U dify dify > /backup/dify_$(date +%Y%m%d).sql`
- ファイルストレージバックアップ: `tar -czf /backup/dify_storage_$(date +%Y%m%d).tar.gz /opt/dify/volumes/app/storage`
- S3アップロード: `aws s3 cp /backup/ s3://dify-workshop-backup/ --recursive`

2. **環境クリーンアップ** (必要に応じて)

**クリーンアップコマンド**
- 一時ファイル削除: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec api flask clean-temp-files`
- ログローテーション: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec nginx nginx -s reopen`

### 6-3. CloudFormation管理手順

#### スタック更新

**スタック更新パラメータ**
- スタック名: dify-workshop-stack
- テンプレートファイル: dify-workshop-infrastructure.yaml
- パラメータ: Environment=workshop
- 権限: CAPABILITY_IAM

**更新コマンド**
- 更新実行: `aws cloudformation update-stack --stack-name dify-workshop-stack --template-body file://dify-workshop-infrastructure.yaml --parameters ParameterKey=Environment,ParameterValue=workshop --capabilities CAPABILITY_IAM`
- 更新完了待機: `aws cloudformation wait stack-update-complete --stack-name dify-workshop-stack`

#### スタック削除（ワークショップ終了後）

**スタック削除パラメータ**
- スタック名: dify-workshop-stack
- 削除対象: 全リソース

**削除コマンド**
- 削除実行: `aws cloudformation delete-stack --stack-name dify-workshop-stack`
- 削除完了待機: `aws cloudformation wait stack-delete-complete --stack-name dify-workshop-stack`

### 6-4. ユーザー管理手順

#### 参加者アカウント作成

**ユーザー作成スクリプト設定**

**基本情報**
- スクリプトパス: /usr/local/bin/create-workshop-users.sh
- ユーザーリストファイル: /opt/dify/config/workshop_users.csv
- デフォルトパスワード: Workshop2025

**処理フロー**
- CSVファイル読み込み
- ヘッダー行スキップ
- ユーザー情報抽出: email, name, role
- flask create-userコマンド実行

**作成コマンド**
- ユーザー作成: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec api flask create-user --email "$email" --name "$name" --role "$role" --password "Workshop2025"`

#### 参加者リスト例

**CSVファイル形式**
- ファイルパス: /opt/dify/config/workshop_users.csv
- フォーマット: email,name,role

**ユーザータイプ別設定例**

**学生アカウント**
- メール: student01@example.com, student02@example.com
- 名前: 学生01, 学生02
- 権限: normal

**講師アカウント**
- メール: instructor@example.com
- 名前: 講師
- 権限: admin

## 7. 障害対応手順

### 7-1. アプリケーション障害

#### 症状: Difyアプリケーション応答なし
**原因調査手順:**

**1. コンテナ状態確認**
- コンテナ一覧: `docker-compose -f /opt/dify/docker-compose.workshop.yml ps`
- APIログ: `docker-compose -f /opt/dify/docker-compose.workshop.yml logs api`
- Webログ: `docker-compose -f /opt/dify/docker-compose.workshop.yml logs web`

**2. リソース使用量確認**
- Dockerスタッツ: `docker stats --no-stream`
- メモリ使用量: `free -h`
- ディスク使用量: `df -h`

**3. ネットワーク接続確認**
- ヘルスチェック: `curl -I http://localhost/health`
- ポート状態: `netstat -tlnp | grep :80`

**復旧手順:**

**1. コンテナ再起動**
- 部分再起動: `docker-compose -f /opt/dify/docker-compose.workshop.yml restart api web`

**2. 全体再起動（必要に応じて）**
- コンテナ停止: `docker-compose -f /opt/dify/docker-compose.workshop.yml down`
- コンテナ起動: `docker-compose -f /opt/dify/docker-compose.workshop.yml up -d`

### 7-2. データベース障害

#### 症状: データベース接続エラー
**原因調査手順:**

**1. PostgreSQLコンテナ状態確認**
- DBログ: `docker-compose -f /opt/dify/docker-compose.workshop.yml logs db`
- DB接続確認: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec db pg_isready -U dify`

**2. データベース容量確認**
- データサイズ: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec db du -sh /var/lib/postgresql/data`

**復旧手順:**

**1. データベース再起動**
- DBコンテナ再起動: `docker-compose -f /opt/dify/docker-compose.workshop.yml restart db`

**2. バックアップからの復旧**
- データ復元: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec db psql -U dify -d dify < /backup/dify_latest.sql`

### 7-3. Bedrock接続障害

#### 症状: AI機能が利用できない
**原因調査手順:**

**1. IAMロール確認**
- 現在の認証情報: `aws sts get-caller-identity`
- IAMロール詳細: `aws iam get-role --role-name DifyWorkshopRole`

**2. Bedrock接続確認**
- 利用可能モデル: `aws bedrock list-foundation-models --region ap-northeast-1`

**3. ネットワーク疎通確認**
- Bedrockエンドポイント: `curl -I https://bedrock-runtime.ap-northeast-1.amazonaws.com`

**復旧手順:**
1. IAMロール再アタッチ
2. セキュリティグループ確認・修正
3. アプリケーション再起動

### 7-4. 性能問題

#### 症状: レスポンス時間が遅い
**調査手順:**

**1. リソース使用量確認**
- CPU使用率: `top`
- I/O状態: `iotop`
- Dockerスタッツ: `docker stats`

**2. ボトルネック特定**
- API応答時間: `curl -w "@curl-format.txt" -o /dev/null -s http://localhost/v1/health`
- DB性能確認: `docker-compose -f /opt/dify/docker-compose.workshop.yml exec db psql -U dify -c "SELECT * FROM pg_stat_activity;"`

**対処手順:**
1. 不要プロセス停止
2. リソース制限調整
3. 必要に応じてインスタンスタイプ変更

---

## 付録

### A. ワークショップチェックリスト

#### 開始前チェック
- [ ] EC2インスタンス起動確認
- [ ] Dockerコンテナ全て起動確認
- [ ] ALBヘルスチェック正常確認
- [ ] Bedrock接続確認
- [ ] 参加者アカウント作成完了
- [ ] 管理者アカウント動作確認

#### 実施中チェック
- [ ] 参加者ログイン状況確認
- [ ] システムリソース監視
- [ ] エラーログ監視
- [ ] 講師サポート体制確認

#### 終了後チェック
- [ ] データバックアップ実行
- [ ] 参加者データ整理
- [ ] システムログ保存
- [ ] 次回準備事項確認

### B. 緊急連絡先

| 役割 | 担当者 | 連絡先 | 対応時間 |
|------|--------|--------|----------|
| システム管理者 | 渡瀬陽己 | xxx-xxxx-xxxx | ワークショップ期間中 |
| ネットワーク管理者 | ネットワークサポート部 | xxx-xxxx-xxxx | 平日9-18時 |
| 講師 | - | xxx-xxxx-xxxx | ワークショップ期間中 |

### C. 参考資料

- [Dify Documentation](https://docs.dify.ai/)
- [Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [AWS CloudWatch User Guide](https://docs.aws.amazon.com/cloudwatch/)