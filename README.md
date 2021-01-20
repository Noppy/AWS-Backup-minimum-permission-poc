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
export REGION=$(aws --output text --profile ${PROFILE} configure get region)
export ACCOUNTID=$(aws --output text --profile ${PROFILE} sts get-caller-identity --query 'Account')
echo -e "PROFILE   = ${PROFILE}\nREGION    = ${REGION}\nACCOUNTID = ${ACCOUNTID}"
```
検証で用意するRoleにAssumeする元のIAMユーザを指定します。
```shell
TRUST_IAMUSER_ARN="<AssumeRole元のIAMユーザARNを指定する>"
```
## (2)KMS CMKの作成(EFSデータバックアップ用の鍵の作成)
```shell
KEY_ID=$( \
aws --profile ${PROFILE} --output text \
    kms create-key \
	    --description "CMK for AWS backup(EFS)" \
	    --origin AWS_KMS \
	--query 'KeyMetadata.KeyId' )

aws --profile ${PROFILE} \
    kms create-alias \
	    --alias-name alias/Key_For_EFSBackup \
	    --target-key-id ${KEY_ID}
```

## (3)IAMロールの作成
### (3)-(a)AWS Backup管理者のIAMロール作成
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
        "backup:CreateBackupVault"
      ],
      "Resource": "arn:aws:backup:'"${REGION}"':'"${ACCOUNTID}"':backup-vault:*"
    }
  ]
}'
#インラインポリシーの設定
aws --profile ${PROFILE} --region ${REGION} \
    iam put-role-policy \
        --role-name "BackupTest-AdminRole" \
        --policy-name "AWSBackupAdminPolicy" \
        --policy-document "${POLICY}";
        
```

## (4)AWS Backup管理者のAWS CLIプロファイル作成
```shell
# BackupTest-AdminRoleのARNを確認する
ADMIN_ROLE_ARN=$(aws --output text --profile ${PROFILE} --region ${REGION} \
    iam get-role \
        --role-name "BackupTest-AdminRole" \
    --query 'Role.Arn')

#Admin用のProfileの設定
echo -e "[profile backupadmin]\nregion = ${REGION}\noutput = json" >> ~/.aws/config
echo -e "[backupadmin]\nrole_arn = ${ADMIN_ROLE_ARN}\nsource_profile = ${PROFILE}" >> ~/.aws/credentials
#設定確認
aws --profile backupadmin sts get-caller-identity

```

AWS Backup Vault作成
```shell
aws --profile backupadmin \
    backup create-backup-vault \
        --backup-vault-name EFS-Test
        --encryption-key-arn 
```
