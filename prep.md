# 準備

## AWS CLI (Command Line Interface) インストール

オンプレミス計算機に、AWSのクラウド計算環境を操作するツール AWS CLI をインストールする。
（参考：[インストール方法の詳細](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html)）

以下は、AWS CLIをオンプレミス計算機のホームディレクトリ下の aws-cliディレクトリにインストールし、
さらに各実行コマンドへのリンクを、そのディレクトリ内の binディレクトリに作成する手順である。

```bash
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ ./aws/install -i ${HOME}/aws-cli -b ${HOME}/aws-cli/bin
```

その後、インストールした AWS CLIのコマンドを実行可能とするため、以下を設定する。
必要に応じて .bashrc等の初期設定ファイルに追加する。

```bash
$ export PATH=${PATH}:${HOME}/aws-cli/bin
```

## アクセスキー作成

アクセスキーは、AWSのアカウントの認証情報である。
AWSのコンソールにログイン後、以下の手順により取得できる。

* コンソール右上のユーザ名をクリックし、セキュリティ認証情報（My Security Credentials）をクリック
* 「アクセスキーの作成」をクリック
* 「.csvファイルのダウンロード」をクリックして保存
* ダウンロードした.csvファイルを開き、アクセスキーとシークレットキーを確認

## AWSプロファイル作成

AWSプロファイルは、AWS CLIコマンドに適用する設定や認証情報をまとめ、一括して指定できるようにしたものである。
オンプレミス計算機で以下のコマンドにより作成する。プロファイル名には、任意の文字列を指定する。

```bash
$ aws configure --profile {プロファイル名}
```

問い合わせに応じて、以下を入力する。
* AWS Access Key ID: アクセスキー
* AWS Secret Access Key: シークレットキー
* Default region name: クラウド上の資源を確保するエリア（例えば ap-northeast-1）
* Default output format: 何も入力しない

その後、以下を実行することにより、これ以降発行するすべての AWS CLIコマンドに、これらの設定が用いられる。

```bash
$ export AWS_PROFILE={プロファイル名}
```
