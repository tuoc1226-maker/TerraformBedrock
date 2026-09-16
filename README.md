# Amazon Bedrock を使用した Terraform RAG テンプレート

[🇯🇵 日本語](README.md) | [🇺🇸 English](README.en.md)

このリポジトリには、Amazon Bedrock 上の埋め込みモデルとして [Amazon Titan V2](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html) を、テキスト生成モデルとして [Claude 3](https://aws.amazon.com/de/bedrock/claude/) を使用した、シンプルな RAG（検索拡張生成）ユースケースの Terraform 実装が含まれています。このサンプルは、以下のユーザーフローに基づいています。

1. ユーザーが Microsoft Excel や PDF ドキュメントなどのファイルを Amazon S3 に手動でアップロードします。（サポートされているファイル形式の詳細については、[Unstructured](https://docs.unstructured.io/open-source/core-functionality/partitioning) のドキュメントを参照してください。）
2. ファイルの内容が抽出され、サーバーレスの [Amazon Aurora with PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html) を基盤とするナレッジデータベースに埋め込まれます。
3. ユーザーがテキスト生成モデルを利用する際、事前にアップロードされたファイルが活用され、検索拡張（retrieval augmentation）によって対話が強化されます。


## アーキテクチャ


![](/media/bedrock-rag-template.drawio_ja.svg)


1. [Amazon S3 バケット](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) `bedrock-rag-template-<account_id>` 内にオブジェクトが作成されるたびに、[Amazon S3 通知](https://docs.aws.amazon.com/AmazonS3/latest/userguide/EventNotifications.html)が [Amazon Lambda 関数](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) `data-ingestion-processor` を呼び出します。 2. Amazon Lambda 関数 `data-ingestion-processor` は、[Amazon ECR リポジトリ](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-iss-ecr.html) `bedrock-rag-template` に格納された Docker イメージに基づいています。この関数は [LangChain S3FileLoader](https://python.langchain.com/v0.1/docs/integrations/document_loaders/aws_s3_file/) を使用してファイルを読み込み、[LangChain Document](https://api.python.langchain.com/en/v0.0.339/schema/langchain.schema.document.Document.html) 形式に変換します。次に、[LangChain RecursiveTextSplitter](https://python.langchain.com/v0.1/docs/modules/data_connection/document_transformers/recursive_text_splitter/) を使用して各ドキュメントをチャンクに分割します。この際、埋め込みモデルである Amazon Titan Text Embedding V2 の最大トークンサイズに基づいた `CHUNK_SIZE` および `CHUNK_OVERLAP` の値が使用されます。続いて、Lambda 関数は [Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) 上の埋め込みモデルを呼び出し、チャンクを数値ベクトル表現に変換（埋め込み）します。最後に、これらのベクトルは Amazon Aurora PostgreSQL データベースに保存されます。Amazon Aurora データベースにアクセスするために、Lambda 関数はまず Amazon Secrets Manager からユーザー名とパスワードを取得します。

3. ユーザーは、[Amazon SageMaker ノートブックインスタンス](https://docs.aws.amazon.com/sagemaker/latest/dg/nbi.html) `aws-sample-bedrock-rag-template` 上で質問プロンプトを作成できます。コードは Amazon Bedrock 上の Claude 3 を呼び出し、ナレッジベースの情報をプロンプトのコンテキストとして提供します。その結果、Claude 3 はドキュメント内の情報を使用して回答を生成します。


### ネットワークとセキュリティ

Amazon Lambda 関数 `data-ingestion-processor` は VPC 内のプライベートサブネットに配置されており、セキュリティグループの設定により、パブリックインターネットへのトラフィック送信は許可されていません。その結果、Amazon S3 および Amazon Bedrock へのトラフィックは VPC エンドポイント経由でのみルーティングされます。これにより、トラフィックがパブリックインターネットを経由しなくなるため、レイテンシーが低減され、ネットワークレベルでのセキュリティが強化されます。

すべてのリソースとデータは、該当する場合、エイリアス `aws-sample/bedrock-rag-template` を持つ Amazon KMS キーを使用して暗号化されます。

このサンプルは任意の AWS リージョンにデプロイ可能ですが、公開時点での Amazon Bedrock における基盤モデルおよび埋め込み（embedding）モデルの利用可能性を考慮し、`us-east-1` または `us-west-1` の使用を推奨します（AWS リージョンごとの Amazon Bedrock 基盤モデルのサポート状況に関する最新リストについては、[AWS リージョン別のモデルサポート](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html)を参照してください）。他の AWS リージョンでこのソリューションを使用する方法については、[次のステップ](#next-steps)のセクションを参照してください。


## 前提条件

### Amazon Web Services

このサンプルを実行するには、有効な AWS アカウントがあり、かつ AWS マネジメントコンソールおよび CLI で十分な権限を持つ IAM ロールにアクセスできることを確認してください。

AWS アカウントの Amazon Bedrock コンソールで、必要な LLM の[モデルアクセスを有効化](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)してください。
この例では、以下のモデルが必要です。

* `amazon.titan-embed-text-v2:0`
* `anthropic.claude-3-sonnet-20240229-v1:0`

### 必要なソフトウェア

このリポジトリをデプロイするには、以下のソフトウェアツールが必要です。

* [Terraform](https://www.terraform.io/):

```shell
❯ terraform --version
Terraform v1.8.4
on linux_amd64
+ provider registry.terraform.io/hashicorp/aws v5.50.0
+ provider registry.terraform.io/hashicorp/external v2.3.3
+ provider registry.terraform.io/hashicorp/local v2.5.1
+ provider registry.terraform.io/hashicorp/null v3.2.2
```

* [Docker](https://docs.docker.com/manuals/)

```shell
❯ docker --version
Docker version 26.0.0, build 2ae903e86c
```

* [Poetry](https://python-poetry.org/)

```shell
❯ poetry --version
Poetry (version 1.7.1)
```

* [Python3.10](https://www.python.org/downloads/release/python-3100/)

* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)


## デプロイ

このセクションでは、インフラストラクチャをデプロイする方法と、Jupyter Notebook でデモを実行する方法について説明します。
> **警告:** 以下の操作を行うと、デプロイ先の AWS アカウントでコストが発生します。


### 認証情報

このサンプルをデプロイするには、[認証情報を環境変数として設定](https://docs.aws.amazon.com/cli/v1/userguide/cli-configure-envvars.html#envvars-set)するか、CLI を直接設定してください。
認証情報が正しく設定されたか確認するには、`aws sts get-caller-identity` を実行します。出力には、現在サインインしているユーザーまたはロールの ARN が含まれているはずです。

### インフラストラクチャ

インフラストラクチャ全体をデプロイするには、以下のコマンドを実行します。

```shell
cd terraform
terraform init
terraform plan -var-file=commons.tfvars
terraform apply -var-file=commons.tfvars
```


### Jupyter Notebook でのデモ

エンドツーエンドのデモは Jupyter Notebook 内で提供されています。以下の手順に従って、ご自身でデモを実行してください。 #### 準備

インフラストラクチャのデプロイにより、VPC内にAmazon SageMakerノートブックインスタンスが作成され、PostgreSQL Auroraデータベースへのアクセス権限が付与されます。インフラストラクチャのデプロイが完了したら、以下の手順に従ってJupyterノートブック上でデモを実行してください。

1. インフラストラクチャがデプロイされたAWSアカウントのAWSマネジメントコンソールにログインします。
2. SageMakerノートブックインスタンス `aws-sample-bedrock-rag-template` を開きます。
3. [rag_demo.ipynb](/rag_demo.ipynb) Jupyterノートブックを、ドラッグ＆ドロップでSageMakerノートブックインスタンスにアップロード（移動）します。
4. SageMakerノートブックインスタンス上で [rag_demo.ipynb](/rag_demo.ipynb) を開き、カーネルとして `conda_python3` を選択します。
5. ノートブックのセルを順次実行してデモを行います。

#### デモの実行

Jupyterノートブックでは、以下のプロセスに沿って作業を進めます。

- 必要なパッケージのインストール
- Embedding（埋め込み）の定義
- データベースへの接続
- データの取り込み（インジェスト）
- RAG（検索拡張生成）によるテキスト生成
- 関連ドキュメントのクエリ実行


### クリーンアップ

インフラストラクチャを削除するには、`terraform destroy -var-file=commons.tfvars` を実行します。


## テスト

### 前提条件 - Python仮想環境

[pyproject.toml](/pyproject.toml) に記載されている依存関係が、Amazon Lambda関数 `data-ingestion-processor` の [requirements](/python/src/handlers/data_ingestion_processor/requirements.txt) と整合していることを確認してください。

依存関係をインストールし、仮想環境を有効化します。

```shell
poetry lock
poetry install
poetry shell

```

### テストの実行


```shell
python -m pytest .
```



## 次のステップ


### 他のAWSリージョンへのデプロイ

このスタックを `us-east-1` および `us-west-1` 以外のAWSリージョンにデプロイするには、2つの方法があります。デプロイ先のAWSリージョンは、[`commons.tfvars`](/terraform/commons.tfvars) ファイルで設定できます。リージョンをまたいで基盤モデル（Foundation Model）にアクセスする場合は、以下のオプションを検討してください。

1. **パブリックインターネットを経由する場合**: トラフィックがパブリックインターネットを経由してもよい場合は、VPCにインターネットゲートウェイを追加します。また、`data-ingestion-processor` という名前の Amazon Lambda 関数および SageMaker ノートブックインスタンスに割り当てられたセキュリティグループを調整し、パブリックインターネットへのアウトバウンドトラフィックを許可するように設定します。
2. **パブリックインターネットを経由しない場合**: このサンプルを `us-east-1` または `us-west-1` 以外の AWS リージョンにデプロイします。`us-east-1` または `us-west-1` リージョンには、`bedrock-runtime` 用の VPC エンドポイントを含む追加の VPC を作成します。次に、VPC ピアリングまたは Transit Gateway を使用して、その VPC とアプリケーション VPC を接続（ピアリング）します。最後に、`us-east-1` または `us-west-1` 以外のリージョンにある AWS Lambda 関数で `bedrock-runtime` 用の boto3 クライアントを設定する際、`us-east-1` または `us-west-1` にある `bedrock-runtime` 用 VPC エンドポイントのプライベート DNS 名を、`endpoint_url` として boto3 クライアントに渡します。VPC ピアリングのソリューションについては、[Terraform AWS VPC Peering](https://github.com/grem11n/terraform-aws-vpc-peering) モジュールを活用できます。

## 依存関係とライセンス

このプロジェクトは MIT ライセンスの下で提供されています。詳細については `LICENSE` ファイルを参照してください。

### 依存関係

* [AWS Lambda Terraform module](https://registry.terraform.io/modules/terraform-aws-modules/lambda/aws/latest)
* [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
* [Terraform](https://developer.hashicorp.com/terraform)
* [Docker Engine](https://docs.docker.com/engine/)


## セキュリティ

詳細については、[CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) を参照してください。

## ライセンス

このライブラリは MIT-0 ライセンスの下でライセンスされています。LICENSE ファイルを参照してください。