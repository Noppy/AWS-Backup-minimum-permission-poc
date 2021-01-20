# AWS-Backup-minimum-permission-poc
AWS Backupの最小権限の調査手順


# 手順
## (1)環境設定
### (1)-(a) 作業環境の準備
下記を準備します。
- bashまたはbash互換が利用可能な環境(LinuxやMacの環境)
- aws-cliのセットアップ
- AdministratorAccessポリシーが付与されたIAMユーザでCLIを実行可能とする、aws-cliのProfileの設定
(1)-(b) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE=<設定したプロファイル名称を指定。デフォルトの場合はdefaultを設定>
export REGION=$(aws --profile ${PROFILE} configure get region)
echo -e "PROFILE = ${PROFILE}\nREGION  = ${REGION}"
```
検証で用意するRoleにAssumeする元のIAMユーザを指定します。
```shell
TRUST_IAMUSER_ARN="<AssumeRole元のIAMユーザARNを指定する>"
```
## (2)IAMロールの作成
### (2)-(a)AWS Backup管理者のIAMロール作成
```shell
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "'"${TRUST_IAMUSER_ARN}"'"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'

#IAMロールの作成
aws --profile ${PROFILE} --region ${REGION} \
    iam create-role \
        --role-name "BackupTest-AdminRole" \
        --assume-role-policy-document "${POLICY}" \
        --max-session-duration 43200


#インラインポリシーの追加
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BackupAdmin",
      "Effect": "Allow",
      "Action": [
        "backup:"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}'
#インラインポリシーの設定
aws --profile ${PROFILE} \
    iam put-role-policy \
        --role-name "BackupTest-AdminRole" \
        --policy-name "AWSBackupAdminPolicy" \
        --policy-document "${POLICY}";
        
```
