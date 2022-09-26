# IAM Exercises

git clone https://github.com/AWSCookbook/Security

export "AWS_REGION=us-east-1"
export "AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)"
export "PRINCIPAL_ARN=$(aws sts get-caller-identity --query Arn --output text)"

RANDOM_STRING=$(aws secretsmanager get-random-password \
--exclude-punctuation --exclude-uppercase \
--password-length 6 --require-each-included-type \
--output text \
--query RandomPassword)

## Creating and Assuming an IAM Role for Deveoper Access

sed -e "s|PRINCIPAL_ARN|${PRINCIPAL_ARN}|g" \
assume-role-policy-template.json > assume-role-policy.json

ROLE_ARN=$(aws iam create-role --role-name AWSCookbook101Role \
--assume-role-policy-document file://assume-role-policy.json \
--output text --query Role.Arn)

aws iam attach-role-policy --role-name AWSCookbook101Role \
--policy-arn arn:aws:iam::aws:policy/PowerUserAccess

aws sts assume-role --role-arn $ROLE_ARN \
--role-session-name AWSCookbook101

## Testing IAM Policy with the IAM Policy Simulator

{
  "Version": "2012-10-17",
  "Statement": [
    {
    "Effect": "Allow",
    "Principal": {
        "Service": "ec2.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
    }
  ]
}

aws iam create-role --role-name AWSCookbook104Role \
--assume-role-policy-document file://assume-role-policy.json --role-name AWSCookbook104Role

aws iam attach-role-policy --role-name AWSCookbook104Role \
--policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

aws iam simulate-principal-policy \
--policy-source-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/AWSCookbook104Role \
--action-names ec2:CreateInternetGateway

aws iam simulate-principal-policy \
--policy-source-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/AWSCookbook104Role \
--action-names ec2:DescribeInstances

## Delegating IAM Administrative Capabilities Using Permissions Boundaries

sed -e "s|PRINCIPAL_ARN|${PRINCIPAL_ARN}|g" \
assume-role-template.json > assume-role.policy.json

ROLE_ARN=$(aws iam create-role --role-name AWSCookbook105Role \
--assume-role-policy-document file://assume-role-policy.json \
--output text --query Role.Arn)

sed -e "s|AWS_ACCOUNT_ID|${AWS_ACCOUNT_ID}|g" \
policy-template.json > policy.json

aws iam create-policy --policy-name AWSCookbook105PB \
--policy-document file://boundary-policy.json

sed -e "s|AWS_ACCOUNT_ID|${AWS_ACCOUNT_ID}|g" \
boundary-policy-template.json > boundary-policy.json

aws iam create-policy --policy-name AWSCookbook105Policy \
--policy-document file://policy.json

aws iam attach-role-policy \
--policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSCookbook105Policy \
--role-name AWSCookbook105Role

creds=$(aws --output text sts assume-role --role-arn $ROLE_ARN \
--role-session-name "AWSCookbook105Role" | \
grep CREDENTIALS | cut -d " " -f2,4,5)

export AWS_ACCESS_KEY_ID=$(echo $creds | cut -d " " -f2)
export AWS_SECRET_ACCESS_KEY=$(echo $creds | cut -d " " -f4)
export AWS_SESSION_TOKEN=$(echo $creds | cut -d " " -f5)

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumRole"
        }
    ]
}

TEST_ROLE_1=$(aws iam create-role --role-name AWSCookbook105Test1 \
--assume-role-policy-document file://lambda-assume-role.policy.json \
--permissions-boundary arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSCookbook105PB \
--output text --query Role.Arn)

aws iam attach-role-policy --role-name AWSCookbook105Test1 \
--policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess

aws iam attach-role-policy --role-name AWSCookbook105Test1 \
--policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess