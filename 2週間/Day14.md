# Day14：AWS CloudFormation を用いた Infrastructure as Code 初めの一歩

## 学習目標

- AWS CloudFormation の基本概念を理解する
- YAML テンプレートを作成して VPC を自動構築する
- 組み込み関数を使用してリソース間の参照を設定する
- パラメータセクションを使用して EC2 インスタンスを構築する
- マッピングセクションを使用してリージョン横断対応を実現する

## 1. Infrastructure as Code (IaC) とは

### 概要

- インフラの構築をコード化し、自動構築を実現する技術
- AWS CloudFormation は AWS における IaC サービス
- YAML または JSON でテンプレートを記述

### CloudFormation の基本用語

- **テンプレート**: AWS リソースの構成をドキュメント形式で記載
- **スタック**: テンプレートから自動構築されたリソースの集合
- **変更セット**: スタックへの変更内容を事前確認する機能

## 2. CloudFormation テンプレートの基本構成

### 主要セクション

1. **Resources セクション** (必須)

   - 作成したい AWS リソースを定義
   - テンプレートの中核となる部分

2. **Parameters セクション**

   - 実行時に埋め込む変数値を定義
   - インスタンスタイプや SSH キーペアなど

3. **Mappings セクション**
   - Map 形式で環境によって変わる値を定義
   - リージョン毎の AMI ID など

### 組み込み関数

- **Ref 関数**: 他のリソースを参照
- **FindInMap**: マッピングセクションの値を参照
- **疑似パラメータ**: AWS::Region など実行環境の情報

## 3. ハンズオン 1: VPC の自動構築

### 事前準備

1. ローカル PC でテキストエディタを起動
2. 新しい YAML ファイルを作成（例：CloudFormation-template.yaml）

### テンプレート作成手順

#### 3.1 基本的な VPC テンプレート

```yaml
Resources:
  CloudFormationVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: CloudFormation-Udemy-AWS-VPC
```

#### 3.2 CloudFormation でスタック作成

1. AWS マネジメントコンソールで CloudFormation サービスに移動
2. 「スタックの作成」をクリック
3. 「テンプレートファイルをアップロード」を選択
4. 作成した YAML ファイルをアップロード
5. スタック名を入力：`udemy-aws-14days-stack`
6. 設定を確認して「送信」

#### 3.3 結果確認

1. VPC コンソールでリソース作成を確認
2. CloudFormation のリソースタブで管理状況を確認

## 4. ハンズオン 2: 組み込み関数とサブネット作成

### 4.1 サブネット追加

```yaml
CloudFormationSubnet:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref CloudFormationVPC
    CidrBlock: 10.0.1.0/24
    AvailabilityZone: ap-northeast-1a
    MapPublicIpOnLaunch: true
    Tags:
      - Key: Name
        Value: CFN-Subnet
```

### 4.2 ネットワークコンポーネント追加

```yaml
# インターネットゲートウェイ
CloudFormationIGW:
  Type: AWS::EC2::InternetGateway
  Properties:
    Tags:
      - Key: Name
        Value: CFN-IGW

# IGWアタッチメント
AttachGateway:
  Type: AWS::EC2::VPCGatewayAttachment
  Properties:
    VpcId: !Ref CloudFormationVPC
    InternetGatewayId: !Ref CloudFormationIGW

# ルートテーブル
CloudFormationRouteTable:
  Type: AWS::EC2::RouteTable
  Properties:
    VpcId: !Ref CloudFormationVPC
    Tags:
      - Key: Name
        Value: CFN-RouteTable

# ルート設定
CloudFormationRoute:
  Type: AWS::EC2::Route
  Properties:
    RouteTableId: !Ref CloudFormationRouteTable
    DestinationCidrBlock: 0.0.0.0/0
    GatewayId: !Ref CloudFormationIGW

# サブネットとルートテーブルの関連付け
SubnetRouteTableAssociation:
  Type: AWS::EC2::SubnetRouteTableAssociation
  Properties:
    SubnetId: !Ref CloudFormationSubnet
    RouteTableId: !Ref CloudFormationRouteTable
```

### 4.3 スタック更新

1. CloudFormation コンソールで「スタックを更新」
2. 変更セットを作成
3. 変更内容を確認して実行

## 5. ハンズオン 3: パラメータセクションと EC2 インスタンス

### 5.1 パラメータセクション追加

```yaml
Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
    Description: EC2 Instance Type

  KeyPair:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Key Name
```

### 5.2 EC2 インスタンス定義

```yaml
CloudFormationEC2Instance:
  Type: AWS::EC2::Instance
  Properties:
    ImageId: ami-xxxxxxxxx # 東京リージョンのAMI ID
    InstanceType: !Ref InstanceType
    KeyName: !Ref KeyPair
    SubnetId: !Ref CloudFormationSubnet
    SecurityGroupIds:
      - !Ref CloudFormationSecurityGroup
    BlockDeviceMappings:
      - DeviceName: /dev/xvda
        Ebs:
          VolumeType: gp3
          VolumeSize: 8
    Tags:
      - Key: Name
        Value: CloudFormation-EC2Instance

CloudFormationSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security Group for CloudFormation
    VpcId: !Ref CloudFormationVPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 0.0.0.0/0
    Tags:
      - Key: Name
        Value: CFN-SecurityGroup
```

### 5.3 パラメータ設定

スタック更新時にパラメータ値を選択：

- インスタンスタイプの選択
- キーペアの選択

## 6. ハンズオン 4: マッピングセクションとリージョン対応

### 6.1 マッピングセクション追加

```yaml
Mappings:
  RegionMap:
    us-east-1:
      x86: ami-xxxxxxxxx # バージニア北部のAMI ID
    ap-northeast-1:
      x86: ami-xxxxxxxxx # 東京リージョンのAMI ID
```

### 6.2 イメージ ID 参照の修正

```yaml
CloudFormationEC2Instance:
  Type: AWS::EC2::Instance
  Properties:
    ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", x86]
    # 他のプロパティは同じ
```

### 6.3 リージョン横断テスト

1. バージニア北部リージョンでキーペア作成
2. 同じテンプレートでスタック作成
3. 異なるリージョンでも同じ構成が作成されることを確認

## 7. スタックの削除

### クリーンアップ手順

1. CloudFormation コンソールで作成したスタックを選択
2. 「削除」をクリック
3. 削除の確認
4. 関連リソースが自動的に削除されることを確認

## 8. CloudFormation ベストプラクティス

### 1. 環境に依存しないテンプレート

- 組み込み関数を積極的に使用
- ハードコーディングを避ける
- アカウント ID やリージョンの依存を排除

### 2. CloudFormation での一元管理

- リソースの追加・変更は必ず CloudFormation で実行
- 手作業での変更を禁止
- テンプレートと実際のリソースの整合性を保持

### 3. バージョン管理の実装

- テンプレートを Git 管理
- プルリクエストによるレビューフロー
- 変更履歴の追跡

### 4. テンプレートの分割

- 更新頻度によるグルーピング
- ネットワークレイヤーとコンピュートレイヤーの分離
- IAM や CloudTrail などの基盤設定の分離

## 9. その他の IaC オプション

### AWS CDK (Cloud Development Kit)

- プログラミング言語でインフラを定義
- Python、JavaScript、TypeScript などに対応
- より柔軟な記述が可能

### AWS Elastic Beanstalk

- 定番構成の自動構築
- アプリケーションデプロイの支援
- 開発に集中したい場合に適している

## まとめ

本日のハンズオンでは、AWS CloudFormation を使用して以下を実現しました：

1. **基本的な VPC 構築**：Resources セクションの使用方法
2. **組み込み関数の活用**：Ref 関数による参照関係の構築
3. **パラメータ化**：実行時の選択肢提供
4. **リージョン対応**：Mappings セクションと疑似パラメータの活用

Infrastructure as Code は、インフラの構築を自動化し、再現性と管理性を向上させる重要な技術です。本日学習した内容を基に、より複雑なシステムの自動化にチャレンジしてください。

---

**注意事項**

- AMI ID は定期的に更新されるため、最新のものを使用してください
- 本番環境では適切なセキュリティ設定を行ってください
- コストを考慮してリソースの削除を忘れずに実行してください
