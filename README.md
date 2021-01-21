# AWS-Backup-minimum-permission-poc
AWS Backupの最小権限の調査手順


# 手順
## (1)環境設定
### (1)-(a) 作業環境の準備
下記を準備します。
- bashまたはbash互換が利用可能な環境(LinuxやMacの環境)
- aws-cliのセットアップ
- AdministratorAccessポリシーが付与されたIAMユーザでCLIを実行可能とする、aws-cliのProfileの設定
- gitコマンド

### (1)-(b) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE=<設定したプロファイル名称を指定。デフォルトの場合はdefaultを設定>
export REGION=$(aws --output text --profile ${PROFILE} configure get region)
export ACCOUNTID=$(aws --output text --profile ${PROFILE} sts get-caller-identity --query 'Account')
echo -e "PROFILE   = ${PROFILE}\nREGION    = ${REGION}\nACCOUNTID = ${ACCOUNTID}"
```

### (1)-(c) gitのclone
VPCやEFS作成用のCloudFormationテンプレートを取得します。
```shell
git clone https://github.com/Noppy/AWS-Backup-minimum-permission-poc.git
cd AWS-Backup-minimum-permission-poc
```

### (1)-(d) パラメータ設定
検証で用意するRoleにAssumeする元のIAMユーザを指定します。
```shell
TRUST_IAMUSER_ARN="<AssumeRole元のIAMユーザARNを指定する>"
```
## (2)KMS CMKの作成(EFSデータバックアップ用の鍵の作成)
```shell
KEY_ID=$( \
aws --profile ${PROFILE} --region ${REGION} --output text \
    kms create-key \
	    --description "CMK for AWS backup(EFS)" \
	    --origin AWS_KMS \
	--query 'KeyMetadata.KeyId' )

aws --profile ${PROFILE} --region ${REGION} \
    kms create-alias \
	    --alias-name alias/Key_For_EFSBackup \
	    --target-key-id ${KEY_ID}
```

## (3)EFSの作成(バックアップ対象の準備)
### (3)-(a) VPCの作成
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableNatGW",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "BackupPoCVPC"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name BackupTest-VPC \
    --template-body "file://./cfns/vpc-4subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```
### (3)-(b) BasionとEFSの作成
```shell
KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。 
AL2_AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;
echo -e "KEYNAME   = ${KEYNAME}\nAL2_AMIID = ${AL2_AMIID}"

# Set Stack Parameters
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AL2_AMIID}"'"
  },
  {
    "ParameterKey": "KEYNAME",
    "ParameterValue": "'"${KEYNAME}"'"
  }
]'
# Create Bastion

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name BackupTest-EfsAndBastion  \
    --template-body "file://./cfns/efs_bastion.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}";
```



### (3)-(b) EFSやBastionの作成


## (4)IAMロールの作成
### (4)-(a)AWS Backup管理者のIAMロール作成
```shell
#CMKのARN取得
CMK_ARN=$(aws --profile ${PROFILE} --region ${REGION} --output text\
    kms describe-key \
        --key-id arn:aws:kms:ap-northeast-1:270025184181:alias/Key_For_EFSBackup \
    --query 'KeyMetadata.Arn' )

#TrustPolicy
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
      "Sid": "ManagedVault",
      "Effect": "Allow",
      "Action": [
        "backup:CreateBackupVault"
      ],
      "Resource": "arn:aws:backup:'"${REGION}"':'"${ACCOUNTID}"':backup-vault:*"
    },
    {
      "Sid": "ManagedBackupStorageForVault",
      "Effect": "Allow",
      "Action": [
        "backup-storage:MountCapsule"
      ],
      "Resource": "*"
    },
    {
      "Sid": "UseCMKForVault",
      "Effect": "Allow",
      "Action": [
          "kms:CreateGrant",
          "kms:GenerateDataKey",
          "kms:Decrypt",
          "kms:RetireGrant",
          "kms:DescribeKey"
       ],
       "Resource": "'"${CMK_ARN}"'"
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

## (5)AWS Backup管理者のAWS CLIプロファイル作成
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

## (6)AWS Backupの設定
### (6)-(a) BackupVault作成
```shell
#CMKのARN取得
CMK_ARN=$(aws --profile ${PROFILE} --region ${REGION} --output text\
    kms describe-key \
        --key-id arn:aws:kms:ap-northeast-1:270025184181:alias/Key_For_EFSBackup \
    --query 'KeyMetadata.Arn' )

#Backup Vaultの作成
aws --profile backupadmin \
    backup create-backup-vault \
        --backup-vault-name TestEFS-BackupVault \
        --encryption-key-arn ${CMK_ARN}
```

### (6)-(b) BackupPlan作成
```shell


{
  "BackupPlanName": "string",
  "Rules": [
    {
      "RuleName": "string",
      "TargetBackupVaultName": "string",
      "ScheduleExpression": "string",
      "StartWindowMinutes": long,
      "CompletionWindowMinutes": long,
      "Lifecycle": {
        "MoveToColdStorageAfterDays": long,
        "DeleteAfterDays": long
      },
      "RecoveryPointTags": {"string": "string"
        ...},
      "CopyActions": [
        {
          "Lifecycle": {
            "MoveToColdStorageAfterDays": long,
            "DeleteAfterDays": long
          },
          "DestinationBackupVaultArn": "string"
        }
        ...
      ]
    }
    ...
  ],
  "AdvancedBackupSettings": [
    {
      "ResourceType": "string",
      "BackupOptions": {"string": "string"
        ...}
    }
    ...
  ]
}
