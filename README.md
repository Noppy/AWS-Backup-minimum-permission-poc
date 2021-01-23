# AWS-Backup-minimum-permission-poc
AWS Backupの最小権限の調査手順

# 参考
- [CLIでAWS BackupのRecoveryPointからEFSを復旧する手順](https://aws.amazon.com/jp/premiumsupport/knowledge-center/aws-backup-restore-efs-file-system-cli/)
- [CLIでAWS BackupのRecoveryPointからEC2を復旧する手順](https://aws.amazon.com/jp/premiumsupport/knowledge-center/aws-backup-cli-create-plan-run-job/)


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
SOURCE_REGION="ap-northeast-1"
DEST_REGION="ap-southeast-2"
```
## (2)KMS CMKの作成(EFSデータバックアップ用の鍵の作成)
### (2)-(a)バックアップ元リージョンでのCMK作成
```shell
# バックアップ元リージョンでのBackupVaultで指定するCMK作成
KEY_ID=$( \
aws --profile ${PROFILE} --region ${SOURCE_REGION} --output text \
    kms create-key \
	    --description "CMK for AWS backup(EFS)" \
	    --origin AWS_KMS \
	--query 'KeyMetadata.KeyId' )

aws --profile ${PROFILE} --region ${SOURCE_REGION} \
    kms create-alias \
	    --alias-name alias/Key_For_EFSBackup \
	    --target-key-id ${KEY_ID}
```
### (2)-(b)バックアップのコピー先リージョンでのCMK作成
```shell
# バックアップのコピー先リージョンでのBackupVaultで指定するCMK作成
KEY_ID=$( \
aws --profile ${PROFILE} --region ${DEST_REGION} --output text \
    kms create-key \
	    --description "CMK for AWS backup(EFS)" \
	    --origin AWS_KMS \
	--query 'KeyMetadata.KeyId' )

aws --profile ${PROFILE} --region ${DEST_REGION} \
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

## (4)IAMロールの作成
### (4)-(a)ロール共通設定
```shell
#CMKのARN取得
SOURCE_CMK_ARN=$(aws --profile ${PROFILE} --region ${SOURCE_REGION} --output text\
    kms describe-key \
        --key-id arn:aws:kms:${SOURCE_REGION}:${ACCOUNTID}:alias/Key_For_EFSBackup \
    --query 'KeyMetadata.Arn' )
DEST_CMK_ARN=$(aws --profile ${PROFILE} --region ${DEST_REGION} --output text\
    kms describe-key \
        --key-id arn:aws:kms:${DEST_REGION}:${ACCOUNTID}:alias/Key_For_EFSBackup \
    --query 'KeyMetadata.Arn' )
echo -e "SOURCE_CMK_ARN = ${SOURCE_CMK_ARN}\nDEST_CMK_ARN   = ${DEST_CMK_ARN}"
#TrustPolicy
TRUST_POLICY='{
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
```

### (4)-(b) AWS Backupサービスでのバックアップ実行用のIAMロール作成
```shell
#TrustPolicy
BACKUP_TRUST_POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "backup.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'

#IAMロールの作成
aws --profile ${PROFILE} --region ${REGION} \
    iam create-role \
        --role-name "BackupTest-ServiceBackupPolicy" \
        --assume-role-policy-document "${BACKUP_TRUST_POLICY}" \
        --max-session-duration 43200

#AWS管理ポリシーのアタッチ
aws --profile ${PROFILE} --region ${REGION} \
    iam attach-role-policy \
        --role-name "BackupTest-ServiceBackupPolicy" \
        --policy-arn "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup"

aws --profile ${PROFILE} --region ${REGION} \
    iam attach-role-policy \
        --role-name "BackupTest-ServiceBackupPolicy" \
        --policy-arn "arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores"
```

### (4)-(c) AWS Backup管理者のIAMロール作成
```shell
#IAMロールの作成
aws --profile ${PROFILE} --region ${REGION} \
    iam create-role \
        --role-name "BackupTest-AdminRole" \
        --assume-role-policy-document "${TRUST_POLICY}" \
        --max-session-duration 43200

#バックアップ実行用のIAMロールのARN取得
BACKUP_SERVICE_ROLE_ARN=$(aws --profile ${PROFILE} --region ${REGION} --output text \
    iam get-role \
        --role-name BackupTest-ServiceBackupPolicy \
    --query 'Role.Arn')
echo -e "BACKUP_SERVICE_ROLE_ARN = ${BACKUP_SERVICE_ROLE_ARN}"

#インラインポリシーの追加
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CreateServiceRole",
      "Effect": "Allow",
      "Action": [
        "iam:CreateServiceLinkedRole"
      ],
      "Resource": [
        "arn:aws:iam::'"${Account}"':role/*"
      ]
    },
    {
      "Sid": "ViewOtherResources",
      "Effect": "Allow",
      "Action": [
        "iam:Get*",
        "iam:List*",
        "elasticfilesystem:Describe*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ReadBackupResouces",
      "Effect": "Allow",
      "Action": [
        "backup:Describe*",
        "backup:Get*",
        "backup:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ManageBackupVaults",
      "Effect": "Allow",
      "Action": [
        "backup:CreateBackupVault",
        "backup:DeleteBackupVault",
        "backup:DeleteBackupVaultAccessPolicy",
        "backup:PutBackupVaultAccessPolicy",
        "backup:DeleteBackupVaultNotification",
        "backup:PutBackupVaultNotifications"
      ],
      "Resource": [
        "arn:aws:backup:*:'"${ACCOUNTID}"':backup-vault:*"
      ]
    },
    {
      "Sid": "ManageBackupStoragesForVaults",
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
      "Resource": [
        "'"${SOURCE_CMK_ARN}"'",
        "'"${DEST_CMK_ARN}"'"
      ]
    },
    {
      "Sid": "ManageBackupPlansAndSelection",
      "Effect": "Allow",
      "Action": [
        "backup:CreateBackupPlan",
        "backup:CreateBackupPlan",
        "backup:UpdateBackupPlan",
        "backup:CreateBackupSelection",
        "backup:DeleteBackupSelection"
      ],
      "Resource": [
        "arn:aws:backup:*:'"${ACCOUNTID}"':backup-plan:*"
      ]
    },
    {
      "Sid": "PassRoleForCreateSelection",
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": [
        "'"${BACKUP_SERVICE_ROLE_ARN}"'"
      ]
    },
    {
      "Sid": "ManagedRecoveryPolint",
      "Effect": "Allow",
      "Action": [
        "backup:DeleteRecoveryPoint"
      ],
      "Resource": [
        "arn:aws:ec2:*::snapshot/*",
        "arn:aws:backup:*:'"${ACCOUNTID}"':recovery-point:*",
        "arn:aws:rds:*:'"${ACCOUNTID}"':snapshot:awsbackup:*",
        "arn:aws:rds:*:'"${ACCOUNTID}"':cluster-snapshot:awsbackup:*"
      ]
    },
    {
      "Sid": "ManageGlobalAndRegionSetting",
      "Effect": "Allow",
      "Action": [
        "backup:UpdateGlobalSettings",
        "backup:UpdateRegionSettings"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EditTagsOfBackupResources",
      "Effect": "Allow",
      "Action": [
        "backup:TagResource",
        "backup:UntagResource"
      ],
      "Resource": "*"
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

### (4)-(d) AWS BackupオペレータのIAMロール作成
```shell
#IAMロールの作成
aws --profile ${PROFILE} --region ${REGION} \
    iam create-role \
        --role-name "BackupTest-OperatorRole" \
        --assume-role-policy-document "${TRUST_POLICY}" \
        --max-session-duration 43200

#バックアップ実行用のIAMロールのARN取得
BACKUP_SERVICE_ROLE_ARN=$(aws --profile ${PROFILE} --region ${REGION} --output text \
    iam get-role \
        --role-name BackupTest-ServiceBackupPolicy \
    --query 'Role.Arn')
echo -e "BACKUP_SERVICE_ROLE_ARN = ${BACKUP_SERVICE_ROLE_ARN}"

#インラインポリシーの追加
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ViewOtherResources",
      "Effect": "Allow",
      "Action": [
        "iam:Get*",
        "iam:List*",
        "elasticfilesystem:Describe*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ReadBackupResouces",
      "Effect": "Allow",
      "Action": [
        "backup:Describe*",
        "backup:Get*",
        "backup:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ExecuteBackupJob",
      "Effect": "Allow",
      "Action": [
        "backup:StartBackupJob",
        "backup:StopBackupJob"
      ],
      "Resource": [
        "arn:aws:backup:*:'"${ACCOUNTID}"':backup-vault:*"
      ]
    },
    {
      "Sid": "ExecuteCopyAndRestoreJob",
      "Effect": "Allow",
      "Action": [
        "backup:StartCopyJob",
        "backup:StartRestoreJob"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "PassRoleForStartXXXXJob",
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": [
        "'"${BACKUP_SERVICE_ROLE_ARN}"'"
      ]
    }
  ]
}'

#インラインポリシーの設定
aws --profile ${PROFILE} --region ${REGION} \
    iam put-role-policy \
        --role-name "BackupTest-OperatorRole" \
        --policy-name "AWSBackupOperatorPolicy" \
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

# BackupTest-OperatorRoleのARNを確認する
OPER_ROLE_ARN=$(aws --output text --profile ${PROFILE} --region ${REGION} \
    iam get-role \
        --role-name "BackupTest-OperatorRole" \
    --query 'Role.Arn')

#OperatorRole用のProfileの設定
echo -e "[profile backupoper]\nregion = ${REGION}\noutput = json" >> ~/.aws/config
echo -e "[backupoper]\nrole_arn = ${OPER_ROLE_ARN}\nsource_profile = ${PROFILE}" >> ~/.aws/credentials
#設定確認
aws --profile backupoper sts get-caller-identity

```

## (6)AWS Backupの設定
### (6)-(a) BackupVault作成
```shell
#CMKのARN取得
SOURCE_CMK_ARN=$(aws --profile ${PROFILE} --region ${SOURCE_REGION} --output text\
    kms describe-key \
        --key-id arn:aws:kms:${SOURCE_REGION}:${ACCOUNTID}:alias/Key_For_EFSBackup \
    --query 'KeyMetadata.Arn' )
DEST_CMK_ARN=$(aws --profile ${PROFILE} --region ${DEST_REGION} --output text\
    kms describe-key \
        --key-id arn:aws:kms:${DEST_REGION}:${ACCOUNTID}:alias/Key_For_EFSBackup \
    --query 'KeyMetadata.Arn' )
echo -e "SOURCE_CMK_ARN = ${SOURCE_CMK_ARN}\nDEST_CMK_ARN   = ${DEST_CMK_ARN}"

#バックアップ元リージョンでの Backup Vaultの作成
aws --profile backupadmin --region ${SOURCE_REGION}\
    backup create-backup-vault \
        --backup-vault-name TestEFS-Source-BackupVault \
        --encryption-key-arn ${SOURCE_CMK_ARN}

#バックアップ先リージョンでの Backup Vaultの作成
aws --profile backupadmin --region ${DEST_REGION}\
    backup create-backup-vault \
        --backup-vault-name TestEFS-Dest-BackupVault \
        --encryption-key-arn ${DEST_CMK_ARN}
```

### (6)-(b) BackupPlan作成
```shell
DEST_BACKUPVAUT_ARN=$(aws --profile backupadmin --region ${DEST_REGION} --output text \
    backup describe-backup-vault \
        --backup-vault-name TestEFS-Dest-BackupVault \
    --query 'BackupVaultArn')

BACKUP_PLAN_JSON='
{
  "BackupPlanName": "TestEFS-testplan",
  "Rules": [
    {
      "RuleName": "HalfDayBackups",
      "TargetBackupVaultName": "TestEFS-Source-BackupVault",
      "ScheduleExpression": "cron(0 5/12 ? * * *)",
      "StartWindowMinutes": 480,
      "CompletionWindowMinutes": 10080,
      "Lifecycle": {
        "MoveToColdStorageAfterDays": 8,
        "DeleteAfterDays": 100
      },
      "RecoveryPointTags": {
        "Key": "Rule",
        "Value": "HalfDayBackups" 
      },
      "CopyActions": [
        {
          "Lifecycle": {
            "MoveToColdStorageAfterDays": 8,
            "DeleteAfterDays": 100
          },
          "DestinationBackupVaultArn": "'"${DEST_BACKUPVAUT_ARN}"'"
        }
      ]
    }
  ]
}
'

#バックアップ元リージョンで Backup Planを作成
aws --profile backupadmin --region ${SOURCE_REGION}\
    backup create-backup-plan \
        --backup-plan "${BACKUP_PLAN_JSON}"
```

### (6)-(c) バックアップ対象のBackupPlanへの登録
バックアップ対象のリソース(ここではEFS)を作成したBackupPlanに登録します。
```shell
#リソース情報の取得
EFS_FILE_SYSTEM_ARN=$(aws --profile backupadmin --region ${SOURCE_REGION} --output text \
    efs describe-file-systems \
    --query 'FileSystems[].{Name:Name,Arn:FileSystemArn}' \
    |grep BackupTest-Volume|awk '{print $1}' )
BACKUP_PLAN_ID=$(aws --profile backupadmin --region ${SOURCE_REGION} --output text \
    backup list-backup-plans \
    --query 'BackupPlansList[].{Name:BackupPlanName,Id:BackupPlanId}' \
    |grep  TestEFS-testplan | awk '{print $1}')
BACKUP_SERVICE_ROLE_ARN=$(aws --profile backupadmin --region ${REGION} --output text \
    iam get-role \
        --role-name BackupTest-ServiceBackupPolicy \
    --query 'Role.Arn')
echo -e "EFS_FILE_SYSTEM_ARN     = ${EFS_FILE_SYSTEM_ARN}\nBACKUP_PLAN_ID           = ${BACKUP_PLAN_ID}\nBACKUP_SERVICE_ROLE_ARN = ${BACKUP_SERVICE_ROLE_ARN}"

#Session用のJSON
BACKUP_SESSIOM_JSON='
{
  "SelectionName": "TestEFSーselection",
  "IamRoleArn": "'"${BACKUP_SERVICE_ROLE_ARN}"'",
  "Resources": [
    "'"${EFS_FILE_SYSTEM_ARN}"'"
  ],
  "ListOfTags": [
    {
      "ConditionType": "STRINGEQUALS",
      "ConditionKey": "aws:elasticfilesystem:custome-backup",
      "ConditionValue": "enabled"
    }
  ]
}'

#Selectionの作成
aws --profile backupadmin --region ${SOURCE_REGION}\
    backup create-backup-selection \
        --backup-plan-id "${BACKUP_PLAN_ID}" \
        --backup-selection "${BACKUP_SESSIOM_JSON}"
```
## (7) オンデマンドでのバックアップ運用
### (7)-(a) オンデマンドでバックアップジョブ実行
定期実行ではなく、手動でワンショットのバックアップジョブを実行する手順です。

```shell
#リソース情報の取得
EFS_FILE_SYSTEM_ARN=$(aws --profile backupoper --region ${SOURCE_REGION} --output text \
    efs describe-file-systems \
    --query 'FileSystems[].{Name:Name,Arn:FileSystemArn}' \
    |grep BackupTest-Volume|awk '{print $1}' )
BACKUP_SERVICE_ROLE_ARN=$(aws --profile backupadmin --region ${REGION} --output text \
    iam get-role \
        --role-name BackupTest-ServiceBackupPolicy \
    --query 'Role.Arn')
UUID=$(python -c 'import uuid; print(uuid.uuid4())' )

echo -e "EFS_FILE_SYSTEM_ARN = ${EFS_FILE_SYSTEM_ARN}\nUUID                = ${UUID}\nBACKUP_SERVICE_ROLE_ARN = ${BACKUP_SERVICE_ROLE_ARN}"

#バックアップジョブの実行
BACKUP_JOB_ID=$(aws --profile backupoper --region ${SOURCE_REGION} --output text \
    backup start-backup-job \
        --idempotency-token ${UUID} \
        --backup-vault-name  "TestEFS-Source-BackupVault" \
        --resource-arn ${EFS_FILE_SYSTEM_ARN} \
        --iam-role-arn ${BACKUP_SERVICE_ROLE_ARN} \
        --start-window-minutes 60 \
        --complete-window-minutes 10080 \
        --lifecycle DeleteAfterDays=30 \
    --query 'BackupJobId' )

#バックアップステータスチェック(COMPLETEDになるまで待機)
while true;
do
    STATUS=$(aws --profile backupoper --region ${SOURCE_REGION} --output text \
        backup describe-backup-job \
            --backup-job-id ${BACKUP_JOB_ID} \
        --query '{Id:BackupJobId,State:State}' )
    echo -e "$(date '+%m/%d %H:%M:%S') ${STATUS}"
    if [ "A$(echo ${STATUS}|awk '{print $2}')" = "ACOMPLETED" ]; then break; fi
done

```

### (7)-(b) 取得したリカバリーポイントを他Regionへコピー
コピー元のリカバリーポイントとコピー先のBackupVaultを指定してバックアップデータをコピーします。
- StartCopyJobは、`コピー元のAWSアカウント&リージョン`で実行する
- コピー元のリカバリーポイントは、StartCopyJobを実行しているを`AWSアカウント&リージョン`の物しか指定できない
  - コピー元のリカバリーポイントは、`格納しているBackupVault名`+`リカバリーポイントARN`の組み合わせで指定する(2つとも指定が必須)
  - `格納しているBackupVault名`は、BackupVault名での指定であり、他アカウント or 他リージョンの物は指定できない
  - したがって、他アカウント or 他リージョンのリカバリーポイントも指定できない。
```shell
#リカバリーポイントのARN取得
RECOVERY_POINT_ARN=$(aws --profile backupoper --region ${SOURCE_REGION} --output text \
        backup describe-backup-job \
            --backup-job-id ${BACKUP_JOB_ID} \
        --query '{RecoveryPointArn:RecoveryPointArn}' )
DEST_BACKUPVAUT_ARN=$(aws --profile backupoper --region ${DEST_REGION} --output text \
    backup describe-backup-vault \
        --backup-vault-name TestEFS-Dest-BackupVault \
    --query 'BackupVaultArn')
BACKUP_SERVICE_ROLE_ARN=$(aws --profile backupoper --region ${REGION} --output text \
    iam get-role \
        --role-name BackupTest-ServiceBackupPolicy \
    --query 'Role.Arn')
UUID=$(python -c 'import uuid; print(uuid.uuid4())' )
echo -e "RECOVERY_POINT_ARN      = ${RECOVERY_POINT_ARN}\nDEST_BACKUPVAUT_ARN     = ${DEST_BACKUPVAUT_ARN}\nBACKUP_SERVICE_ROLE_ARN = ${BACKUP_SERVICE_ROLE_ARN}\nUUID = ${UUID}"

#リカバリーポイントの他Regionコピー
COPY_JOB_ID=$(aws --profile backupoper --region ${SOURCE_REGION} --output text \
    backup start-copy-job \
        --recovery-point-arn ${RECOVERY_POINT_ARN} \
        --source-backup-vault-name "TestEFS-Source-BackupVault" \
        --destination-backup-vault-arn ${DEST_BACKUPVAUT_ARN} \
        --iam-role-arn ${BACKUP_SERVICE_ROLE_ARN} \
        --idempotency-token ${UUID} \
        --lifecycle DeleteAfterDays=30 \
    --query 'CopyJobId' )

#バックアップステータスチェック(COMPLETEDになるまで待機)
while true;
do
    STATUS=$(aws --profile backupoper --region ${SOURCE_REGION} --output text \
        backup describe-copy-job \
            --copy-job-id ${COPY_JOB_ID} \
        --query 'CopyJob.{Id:CopyJobId,State:State}' )
    echo -e "$(date '+%m/%d %H:%M:%S') ${STATUS}"
    if [ "A$(echo ${STATUS}|awk '{print $2}')" = "ACOMPLETED" ]; then break; fi
    sleep 10
done
```

## (8) リストア運用
```shell
#データ手動設定
RESTORE_RECOVERY_POINT_ARN="<復元元のリカバリーポイントのARNを指定する>"
RESTORE_BACKUP_VAULT="<復元元のリカバリーポイントが格納されているバックアップボルト名を指定>"
RESTORE_STORED_REGION="<復元元のリカバリーポイントが格納されるREGIONを指定>"
RESTORE_RESTORE_REGION="<復元先のリージョンを指定>"
RESTORE_CMK="${DEST_CMK_ARN}"

#データ取得/生成
FILESYSTEM_ID=$(aws --profile backupoper --region ${RESTORE_STORED_REGION} --output text \
    backup get-recovery-point-restore-metadata \
        --backup-vault-name ${RESTORE_BACKUP_VAULT} \
        --recovery-point-arn ${RESTORE_RECOVERY_POINT_ARN} \
    --query 'RestoreMetadata."file-system-id"' )
BACKUP_SERVICE_ROLE_ARN=$(aws --profile backupoper --region ${REGION} --output text \
    iam get-role \
        --role-name BackupTest-ServiceBackupPolicy \
    --query 'Role.Arn')
UUID=$(python -c 'import uuid; print(uuid.uuid4())' )

#リストア実行
METADATA_JSON='{
  "file-system-id": "'"${FILESYSTEM_ID}"'",
  "Encrypted": "true",
  "KmsKeyId": "'${RESTORE_CMK}'",
  "PerformanceMode": "generalPurpose", 
  "CreationToken": "'"${UUID}"'", 
  "newFileSystem": "true"
}'

aws --profile backupoper --region ${RESTORE_RESTORE_REGION} \
    backup start-restore-job \
        --recovery-point-arn ${RESTORE_RECOVERY_POINT_ARN} \
        --metadata "${METADATA_JSON}" \
        --iam-role-arn ${BACKUP_SERVICE_ROLE_ARN} \
        --idempotency-token ${UUID} \
        --resource-type "EFS"

```
