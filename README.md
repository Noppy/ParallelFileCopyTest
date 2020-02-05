# SGWPoC-ParallelFileCopyTest
CSVリストに従ってパラレルでファイルコピーを実行し実行時間を測定するpythonツール
# Precondition
## (a)File Gateway
- instance type: c5.4xlarge
- ebs:
    - rootvol: io1, 240GiB
    - datavol: io1, 16TiB 3000IOPS

## (b)Windows Client
- Windows Instance
    - instance size: m5ad.4xlarge
- [python3 for windows](https://www.python.org/downloads/windows/)
- [git for windows ](https://gitforwindows.org/) (Used to install verification tools)



# Setup
## (a) File Gateway
Prerequisite: File gateway has been created and activated.
### Set SMB guest password
```sh
GATEWAY_ARN=<REPLACE_WITH_THE_COLLECT_GW_ARN>
PASSWORD="HogeHoge@"

aws --profile ${PROFILE} storagegateway \
    set-smb-guest-password \
        --gateway-arn ${GATEWAY_ARN} \
        --password ${PASSWORD}
```
### Create a File share
```sh
# Set parameters
BUCKETARN="<REPLACE_WITH_THE_CORRECT_BUCKET_ARN>"
ROLEARN="<REPLACE_WITH_THE_CORRECT_IAM_ROLE_ARN_FOR_File_Gateway>"
GATEWAY_ARN="<REPLACE_WITH_THE_COLLECT_GW_ARN>"
CMK_ARN="<REPLACE_WITH_THE_COLLECT_CMK_ARN>"

CLIENT_TOKEN=$(cat /dev/urandom | base64 | fold -w 38 | sed -e 's/[\/\+\=]/0/g' | head -n 1)

# Create a file share
aws --profile ${PROFILE} storagegateway \
    create-smb-file-share \
        --client-token ${CLIENT_TOKEN} \
        --gateway-arn "${GATEWAY_ARN}" \
        --location-arn "${BUCKETARN}" \
        --role "${ROLEARN}" \
        --object-acl bucket-owner-full-control \
        --default-storage-class S3_STANDARD \
        --guess-mime-type-enabled \
        --authentication GuestAccess \
        --kms-encrypted \
        --kms-key ${CMK_ARN} ;
```
## (b) Windows Client
- install python3 and git

```sh
git clone https://github.com/Noppy/ParallelFileCopyTest.git
```

# 使い方
## ファイルリストの作成(copy_list.csv)
CSV形式で、タスクで実行するときに指定する、コピー元ファイル(2個)とコピー先ディレクトリを記載します。  
1行、1タスク分で、100タスク実行する場合は100行記載します。
```
"最初のコピーの元ファイル", "2番目のコピーの元ファイル", "コピー先ディレクトリ"
"F:/20150103/0000/test-001000KB_000300.dat","F:/20150103/0000/test-001000KB_000800.dat","D:/"
"F:/20150103/0001/test-001000KB_000301.dat","F:/20150103/0001/test-001000KB_000801.dat","D:/"
"F:/20150103/0002/test-001000KB_000302.dat","F:/20150103/0002/test-001000KB_000802.dat","D:/"
"F:/20150103/0003/test-001000KB_000303.dat","F:/20150103/0003/test-001000KB_000803.dat","D:/"
```

## 実行
```sh
 python.exe .\FileCopyTEst.py .\copy_list.csv
 ```

## 実行結果
実行結果は、下記に記録されます。
- ResultsSummary.csv : 実行の結果サマリーが記録されます。
- ResultsDetail__XXXX_YYYYMMDD_HHMMSS.csv : 各タスクの実行結果が記録された詳細結果です。XXXXは同時実行です。

### ResultsSummary.csvのフィールド
- タスク数
- 同時実行数(タスク数と同じ)
- 全体の実行時間(sec)
- 全体の開始時間
- 全体の終了時間
- 成功したタスク数
- 失敗したタスク数
- 結果不明なタスク数
- 実行結果のタスク合計数

### ResultsDetail__XXXX_YYYYMMDD_HHMMSS.csvのフィールド
- 最初のコピー元ファイル
- 2番目のコピー元ファイル
- コピー先
- タスクの開始時間
- テストプログラム(親プログラム)開始時間を基準とした、最初のファイルのコピー開始までの時間(msec)
- テストプログラム(親プログラム)開始時間を基準とした、最初のファイルのコピー完了までの時間(msec)
- テストプログラム(親プログラム)開始時間を基準とした、2番目のファイルのコピー完了までの時間(msec)
- タスクの結果(Success or Failed)
- タスクがFailedになった時の、エラーメッセージ
