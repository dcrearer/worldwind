# S3 Exercises

git clone https://github.com/AWSCookbook/Storage

export "AWS_REGION=us-east-1"
export "AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)"
export "PRINCIPAL_ARN=$(aws sts get-caller-identity --query Arn --output text)"

RANDOM_STRING=$(aws secretsmanager get-random-password \
--exclude-punctuation --exclude-uppercase \
--password-length 6 --require-each-included-type \
--output text \
--query RandomPassword)

## Lifecycle Management

aws s3api create-bucket --bucket awscookbook301-$RANDOM_STRING

aws s3api put-bucket-lifecycle-configuration \
--bucket awscookbook301-$RANDOM_STRING \
--lifecycle-configuration file://lifecycle-rule.json

aws s3api get-bucket-lifecycle-configuration \ 
--bucket awscookbook301-$RANDOM_STRING  

aws s3 cp book_cover.png s3://awscookbook301-$RANDOM_STRING

aws s3api list-objects-v2 --bucket awscookbook301-$RANDOM_STRING

## Intelligent-Tiering Archive Policies to Automatically Archive S3 Objects

aws s3api create-bucket --bucket awscookbook302-$RANDOM_STRING

aws s3api put-bucket-intelligent-tiering-configuration \
--bucket awscookbook302-$RANDOM_STRING \
--id awscookbook302 \
--intelligent-tiering-configuration "$(cat tiering.json)"

aws s3api get-bucket-intelligent-tiering-configuration \
--bucket awscookbook302-$RANDOM_STRING \
--id awscookbook302

aws s3 cp book_cover.png s3://awscookbook302-$RANDOM_STRING

aws s3api list-objects-v2 --bucket awscookbook302-$RANDOM_STRING

## Replicating S3 Bucket to Meet Recovery Point Objectives

aws s3api create-bucket --bucket awscookbook303-src-$RANDOM_STRING

aws s3api put-bucket-versioning \
--bucket awscookbook303-src-$RANDOM_STRING \
--versioning-configuration Status=Enabled

test -d .venv || python3 -m venv .venv
source .venv/bin/activate
pip install boto3

aws s3api create-bucket --bucket awscookbook303-dst-$RANDOM_STRING

aws s3api put-bucket-versioning \
--bucket awscookbook303-dst-$RANDOM_STRING \
--versioning-configuration Status=Enabled

ROLE_ARN=$(aws iam create-role --role-name AWSCookbook303S3Role \
--assume-role-policy-document file://s3-assume-role-policy.json \
--output text --query Role.Arn)

sed -e "s/DSTBUCKET/awscookbook303-dst-${RANDOM_STRING}/g" \
-e "s|SRCBUCKET|awscookbook303-src-${RANDOM_STRING}|g" \
s3-perms-policy-template.json > s3-perms-policy.json

aws iam put-role-policy \
--role-name AWSCookbook303S3Role \
--policy-document file://s3-perms-policy.json \
--policy-name S3ReplicationPolicy

sed -e "s|ROLEARN|${ROLE_ARN}|g" \
-e "s|DSTBUCKET|awscookbook303-dst-${RANDOM_STRING}|g" \
s3-replication-template.json > s3-replication.json

aws s3api put-bucket-replication \
--replication-configuration file://s3-replication.json \
--bucket awscookbook303-src-${RANDOM_STRING} 

aws s3api get-bucket-replication \
--bucket awscookbook303-src-${RANDOM_STRING} 

aws s3 cp book_cover.png s3://awscookbook303-src-${RANDOM_STRING} 

aws s3api head-object --bucket awscookbook303-src-${RANDOM_STRING} \
--key book_cover.png

