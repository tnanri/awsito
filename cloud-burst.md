# オンプレミス計算機からクラウド計算環境へのジョブ投入

クラウド計算環境上に計算クラスタを構築し、オンプレミス計算機から利用する。
計算クラスタにはジョブスケジューラSlurmを起動させ、オンプレミス計算機からの要求に応じてジョブを自動投入する。
また、ジョブに必要なファイルの計算クラスタへの送信や、ジョブの結果の格納には、S3バケットを利用する。
ジョブの自動投入の仕組みには、AWSのイベント検知機能である AWS EventBridgeを用いる。
これにより、ジョブ投入用のS3バケットへのファイル追加を検知してジョブ投入コマンドを自動発行する仕組みを実装する。

## AWS ParallelClusterのインストール

AWSの計算クラスタ作成ツール ParallelClusterをインストールする。（参考：[ParallelClusterインストールの詳細](https://docs.aws.amazon.com/parallelcluster/latest/ug/install-v3-pip.html)）
なお、このツールの利用にはPython 3.7以上が必要である。
また、ParallelClusterが使用する NVM (Node Version Manager) と、Node.js もインストールする。

オンプレミス計算機で、以下を実行する。
```bash
$ python3 -m pip install "aws-parallelcluster" --upgrade --user
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
$ chmod ug+x ~/.nvm/nvm.sh
$ source ~/.nvm/nvm.sh
$ nvm install --lts
```

その後、以下で Node.jsのバージョンを確認する。
```bash
$ node --version
```
もしここでエラーが発生した場合、以下のコマンドにより NVMのバージョンを変更する。
```bash
$ nvm install 17
$ nvm use 17
```

## ジョブ投入用S3バケット作成

AWSの計算クラスタにジョブを投入する際、必要なファイルを送信したり、
ジョブの結果を格納するために利用するS3バケットを作成する。
```bash
$ aws s3 mb s3://{ジョブ投入用バケット名}
```

## 計算クラスタ作成

### キーペア作成

作成する計算クラスタの入り口となるHead NodeにSSHログインするためのキーペアを作成する。
```bash
$ aws ec2 create-key-pair --key-name {キーペア名} --query 'KeyMaterial' --output text > {秘密鍵ファイル名}
$ chmod 400 {秘密鍵ファイル名}
```

### VPC (Virtual Private Cloud) およびサブネットの作成

計算クラスタが利用するネットワーク環境を作成する。
ここで出力される VPC IDを以降のコマンドで使用するので、メモしておく。
```bash
$ aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text
```

VPC内にサブネットを作成する。{VPC ID}には、VPC作成時に表示された VPC IDを入力する。
ここで出力されるサブネットIDを以降のコマンドで使用するので、メモしておく。
```bash
$ aws ec2 create-subnet --vpc-id {VPC ID} --cidr-block 10.0.1.0/24 --query Subnet.SubnetId --output text
```

### インターネットへの接続設定

計算クラスタからインターネットにアクセスするためのインターネットゲートウェイを作成する。
ここで出力されるインターネットゲートウェイIDを以降のコマンドで使用するので、メモしておく。
```bash
$ aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text
```

インターネットゲートウェイを VPCにアタッチする。
```bash
$ aws ec2 attach-internet-gateway --internet-gateway-id {インターネットゲートウェイ ID} --vpc-id {VPC ID}
```

VPCに設定されたルートテーブルのIDを以下で表示し、メモしておく。
```bash
$ aws ec2 describe-route-tables --filters Name=vpc-id,Values={VPC ID} --query "RouteTables[].RouteTableId" --output text
```

インターネットゲートウェイにつながるルートを、ルートテーブルに追加する。
```bash
$ aws ec2 create-route --route-table-id {ルートテーブル ID} --destination-cidr-block 0.0.0.0/0 --gateway-id {インターネットゲートウェイ ID}
```

VPCのDNSホスト名を有効化する。
```bash
$ aws ec2 modify-vpc-attribute --vpc-id {VPC ID} --enable-dns-hostnames '{"Value":true}' 
```

### 計算クラスタ設定ファイルの作成

計算クラスタの作成に使う設定ファイルを作成する。
```bash
$ pcluster configure --config {クラスタ設定ファイル名}
```

問い合わせに応じて、以下を入力する。
なお、後述する PlacementGroup機能を使って単一の領域の中で全ての計算ノードを確保する場合、
計算ノードとして使用する計算機のインスタンスの選択肢に制限がある。（[https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/placement-groups.html#concepts-placement-groups](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/placement-groups.html#concepts-placement-groups)）

| 項目 | 入力内容 |
|:----|:---------|
| Region ID | AWSプロファイルと同じもの（例えば ap-northeast-1）|
| EC Key Pair Name | 作成したキーペアの名前 |
| Scheduler | ジョブスケジューラ（例えば Slurm）|
| Operating System | 利用を希望するOS（例えば ubuntu2004）|
| Head node instance type | Head Nodeとして使用する計算機のインスタンス（例えば t3a.small）|
| Number of queues | ジョブキューの数（例えば 1）|
| Name of queue 1 | ジョブキューの名前（例えば queue1）|
| Number of compute resources for compute | 計算ノードの種類の数（例えば 1）|
| Compute node instance type| 計算ノードとして使用する計算機のインスタンス（例えばc4.large）|
| Maximum instance count | 使用する計算ノード数の最大値（例えば 10）|
| Automate VPC creation? | VPCの自動生成（nを選択し、前述の VPC IDを入力）|
| Automate subnet creation | サブネットの自動生成（nを選択し、前述のサブネットIDを入力）|
| Availability zone | Head Nodeや計算ノードを確保する領域名（例えば ap-northeast-1a）|
| Network configuration | 計算クラスタのネットワーク構成（例えば 1 (Head Nodeのみパブリックネットワーク)）|



### 計算クラスタ設定ファイルの編集

作成されたクラスタ設定ファイルをテキストエディタで開き、以下を修正する。

#### Head Nodeへのインターネットからのアクセス許可
以下のように、ElasticIpを trueに設定する。
```
HeadNode:
  InstanceType: t3a.small
  Networking:
    ElasticIp: true
    SubnetId: {サブネット ID}
```
#### 計算ノードからインターネットへのアクセス許可
以下のように、AssignPublicIpをtrueに設定する。
```
Scheduling:
  ...
  Networking:
    SubnetIds:
    - {サブネット ID}
    AssignPublicIp: true
```

#### 単一の領域の中で全ての計算ノードを確保
以下のように、PlacementGroupの Enabledを trueに設定する。
```
Scheduling:
  SlurmQueues:
    ...
    Networking:
      SubnetIds:
      - {サブネット ID}
      PlacementGroup:
         Enabled: true
```

#### ジョブの自動投入のためのアクセス権設定
計算クラスタでジョブを自動投入する機能のため、
以下のように、Iamの項目を追記する。
```
HeadNode:
  ...
  Iam:
    S3Access:
      - BucketName: {ジョブ投入用バケット名}
        EnableWriteAccess: true
    AdditionalIamPolicies:
      - Policy: arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

### 計算クラスタ作成
以下のコマンドにより、計算クラスタの作成を開始する。
```bash
$ pcluster create-cluster --cluster-name {クラスタ名} --cluster-configuration {クラスタ設定ファイル名}
```

計算クラスタの作成にはしばらく時間を要する。
以下により、作成の状況を確認できる。
```bash
$ pcluster describe-cluster --cluster-name {クラスタ名}
```

計算クラスタの作成が完了すると、cloudFormationStackStatusが CREATE_COMPLETEになる。
この際 instanceIdとして表示される文字列を後で使用するのでメモしておく。


### ジョブ投入用バケットからのファイル変更通知の設定
以下のコマンドにより、ジョブ投入用パケットへのファイル追加発生を AWS EventBridgeに通知するよう設定する。
```bash
$ aws s3api put-bucket-notification-configuration --bucket {ジョブ投入用バケット名} --notification-configuration '{ "EventBridgeConfiguration": {} }'
```

### ジョブ自動投入スクリプト、及びエピローグスクリプトの作成
まず、以下のコマンドで計算クラスタの Head Nodeにログインする。
```bash
$ pcluster ssh --cluster-name {クラスタ名} -i {秘密鍵ファイル名}
```

その後、vimなどのテキストエディタで /home/ubuntu/sub-job.sh という名前のテキストファイルを新規に開き、
以下の内容を入力して保存する。（{ジョブ投入バケット名}には、作成したバケットの名前を記入する。）
```
#!/bin/bash

S3_JOB_BUCKET_NAME={ジョブ投入用バケット名}

ITO_WORK_DIR=/home/ubuntu/job-aws
JOBSTAT=job-aws-stat.txt
JOBSCRIPT=job-aws-run.sh
SUBMITOUT=job-aws-sbatch.out
JOBID=job-aws-jobid.txt

mkdir -p "${ITO_WORK_DIR}" && cd "${ITO_WORK_DIR}"
aws s3 sync "s3://${S3_JOB_BUCKET_NAME}" ./

for d in $(ls -d */)
do
    if [[ -f "${d}${JOBSTAT}" ]]
    then
        STAT=$(head -n 1 "${d}${JOBSTAT}")
        case "${STAT}" in
            "FILE_UPLOAD")
                if [[ -f "${d}${JOBSCRIPT}" ]]
                then
                    (cd "${d}"; /opt/slurm/bin/sbatch "${JOBSCRIPT}" >& "${SUBMITOUT}")
                    if [[ -n $(grep "^sbatch: error:" "${d}${SUBMITOUT}") ]]
                    then
                        echo "JOB_ERROR" > "${d}${JOBSTAT}"
                    else
                        echo "JOB_SUBMIT" > "${d}${JOBSTAT}"
                        awk '{print $4}' "${d}${SUBMITOUT}" > "${d}${JOBID}"
                    fi
                    aws s3 cp "./${d}${JOBSTAT}" "s3://${S3_JOB_BUCKET_NAME}/${d}" 
                fi
                ;;
        esac
    fi
done
```

同様に、/home/ubuntu/epilog.shという名前のテキストファイルを新規に開き、以下の内容を入力して保存する。
```
#!/bin/bash

S3_JOB_BUCKET_NAME={ジョブ投入用バケット名}

ITO_WORK_DIR=/home/ubuntu/job-aws

JOBSTAT=job-aws-stat.txt
JOB_PREFIX=$(basename "${SLURM_JOB_WORK_DIR}")

cd "${SLURM_JOB_WORK_DIR}"
echo "JOB_COMPLETE" > "${JOBSTAT}"

cd ..
aws s3 sync "./${JOB_PREFIX}" "s3://${S3_JOB_BUCKET_NAME}/${JOB_PREFIX}"
```

さらに、以下のコマンドにより、それぞれのファイルに実行権を与える。
```bash
$ chmod +x sub-job.sh epilog.sh
```

その後、以下のコマンドで Slurmの設定ファイルを開く。
```bash
$ sudo vim /opt/slurm/etc/slurm.conf
```

設定ファイル中の SlurmUserを rootに変更し、さらに以下の行を追加する。
```
EpilogSlurmctld=/home/ubuntu/epilog.sh
```

vimを終了後、以下のコマンドで Slurmを再起動する。

```bash
$ sudo systemctl restart slurmctld.service
```

### イベントルール作成
AWSコンソールにログイン後 EventBridgeを検索する。
その後、以下の手順でイベントルールを作成する。
* 「ルールを作成」をクリック
* 名前に任意の文字列を入力
* 「次へ」をクリック
* 「イベントパターン」に以下を入力
| 項目 | 入力内容|
|:-----|:-------|
| イベントソース | AWSのサービス |
| AWSのサービス | Simple Storage Service (S3) |
| イベントタイプ | Amazon S3 イベント通知 |
| 特定のイベント | Object Created |
| 特定のバケット | ジョブ投入用バケット名 |
* 「次へ」をクリック
* 「ターゲット１」に以下を入力
| 項目 | 入力内容|
|:-----|:-------|
| ターゲットを選択 | System Manager 実行コマンド |
| ドキュメント |   AWS-RunShellScript |
| ターゲットキー |  InstanceIds  |
| ターゲット値  |  作成した計算クラスタの instanceId  |
| 自動化パラメータ  | 「定数」をチェックし、以下を入力してそれぞれ「追加」をクリック  |
|   |  Commands : /home/ubuntu/sub-job.sh  |
|   |  Working Directory : /home/ubuntu  |
|   |  Execution Timeout :  sub-job.shの最長実行時間（秒）（例えば 600） |
* 「次へ」をクリック
* タグには何も入力せず、「次へ」をクリック
* 「ルールの作成」をクリック

## ジョブの投入

### プレフィクスを決める
S3バケットにファイルをコピーする際に使用するプレフィクスとして、ジョブ毎に異なる名前（例えば job0001 など）を決めておく。

### ジョブスクリプトの作成
オンプレミス計算機で、job-aws-run.sh という名前で Slurmのジョブスクリプトを作成する。
この中で、プログラムのコンパイルや実行など、ジョブとして実行したいコマンドを記述する。

### ジョブに必要なファイルのコピー
ジョブに必要なファイルをオンプレミス計算機からジョブ投入用S3バケットにジョブのプレフィクスを付けてコピーする。

例えば hello.c のファイルは、以下のコマンドでコピーできる。
```bash
$ aws s3 cp ./hello.c s3://{ジョブ投入用バケット名}/{ジョブのプレフィクス}/
```

また、workディレクトリの全てのファイルを一括してコピーする場合、以下のように syncを利用する。
```bash
$ aws s3 sync ./work s3://{ジョブ投入用バケット名}/{ジョブのプレフィクス}/
```

job-aws-run.sh も、同様にコピーする。

### ジョブの状態通知オブジェクトをジョブ投入用S3バケットにコピー
ジョブに必要なファイルのコピーが完了したら、テキストエディタで job-aws-stat.txt という名前のファイルを開き、以下の内容を記述して保存する。
```
FILE_UPLOAD
```

このファイルを、ジョブのプレフィクスを付けてジョブ投入用S3バケットにコピーする。
```bash
$ aws s3 cp ./job-aws-stat.txt s3://{ジョブ投入用バケット名}/{ジョブのプレフィクス}/
```

## ジョブの状態確認
ジョブの状態は、以下のコマンドでジョブの状態通知オブジェクトを取得し、表示することで確認できる。
```bash
$ aws s3 cp s3://{ジョブ投入用バケット名}/{ジョブのプレフィクス}/job-aws-stat.txt .
$ cat job-aws-stat.txt
```

ジョブの状態通知オブジェクトの内容が JOB_SUBMITならジョブ完了待ち、JOB_COMPLETEならジョブ完了を示す。

## ジョブの結果取得
ジョブの状態がJOB_COMPLETEになったら、以下で作成されたオブジェクトを確認できる。
```bash
$ aws s3 ls s3://{ジョブ投入用バケット名}/{ジョブのプレフィクス}/
```

その後、必要なオブジェクトをダウンロードする。

