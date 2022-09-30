# AWS Setup

## Install Plugin

```shell
pip3 install certbot-dns-route53
```

## Hosted Zone Creation

Create a hosted zone. Parameters required:

* $DOMAIN

```shell
aws route53 create-hosted-zone --name $DOMAIN --caller-reference $(date +"%Y-%m-%dT%H:%M:%S%z")
```

Take note of the `NameServers` for use in your domain registrar.

## User, Policy and Role Creation

Create a new user.

```shell
aws iam create-user --user-name certbot-dns-route53
```

Take note of the `ARN` for use in the trust policy file below.

Create a policy file. Parameters required:

* $HOSTED_ZONE_ID

```json
{
  "Version": "2012-10-17",
  "Id": "certbot-dns-route53",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:GetChange"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/$HOSTED_ZONE_ID"
      ]
    }
  ]
}
```

Create a policy. Parameters required:

* $POLICY_FILE_PATH

```shell
aws iam create-policy --policy-name certbot-dns-route53 --policy-document file://$POLICY_FILE_PATH
```

Take note of the `ARN` for use in the command to attach it to the user.

Attach the policy to the user. Parameters required:

* $POLICY_ARN

```shell
aws iam attach-user-policy --user-name certbot-dns-route53 --policy-arn "$POLICY_ARN"
```

Create a trust policy file. Parameters required:

* $USER_ARN

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {
      "AWS": "$USER_ARN"
    },
    "Action": "sts:AssumeRole"
  }
}
```

Create a role, attach it to a policy and create an access key. Parameters required:

* $TRUST_POLICY_FILE_PATH
* $POLICY_ARN

```shell
aws iam create-role --role-name certbot-dns-route53 --assume-role-policy-document file://$TRUST_POLICY_FILE_PATH

aws iam attach-role-policy --role-name certbot-dns-route53 --policy-arn "$POLICY_ARN"

aws iam create-access-key --user-name certbot-dns-route53
```

Take note of the `AccessKeyId` and `SecretAccessKey` and set environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

## Certificate Creation

Create your certificate. Parameters required:

* $INI_FILE_PATH
* $DOMAIN - domain used in your hosted zone

```shell
sudo certbot certonly \
  --dns-route53 \
  -d "$DOMAIN"
```



# Azure Setup

## Install Plugin

```shell
pip3 install certbot-dns-azure
```

## Service Principal Creation

Create a new service principal. Parameters required:

* $SUBSCRIPTION_ID
* $RESOURCE_GROUP_ID

```shell
az ad sp create-for-rbac \
--name certbot-dns-azure \
--role "DNS Zone Contributor" \
--scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP_ID
```

Take note of the `appId` (aka "client id"), `password` (aka "client secret") and `tenant` for use in the ini file below.

```json
{
  "appId": "a2dec69a-83b1-4db6-803a-e4ea2bfccfa1",
  "displayName": "certbot-dns-azure",
  "password": "swA8Q~r4wNY~lSUVFmSClPS44I8MK6dWnAt39cYZ",
  "tenant": "12248f74-371f-4cc2-9a01-c62a0577a0c2"
}
```

## Certbot Azure Config

Create the certbot Azure config file (filetype ini). Parameters required:

* $CLIENT_ID
* $CLIENT_SECRET
* $TENTANT_ID
* $DOMAIN - domain used in your hosted zone
* $DNS_ZONE_RESOURCE_GROUP_ID - resource group containing the hosted zone

```ini
dns_azure_sp_client_id = $CLIENT_ID
dns_azure_sp_client_secret = $CLIENT_SECRET
dns_azure_tenant_id = $TENTANT_ID

dns_azure_environment = "AzurePublicCloud"

dns_azure_zone1 = $DOMAIN:$DNS_ZONE_RESOURCE_GROUP_ID
```

## Certificate Creation

Create your certificate. Parameters required:

* $INI_FILE_PATH
* $DOMAIN - domain used in your hosted zone

```shell
sudo certbot certonly \
  --authenticator dns-azure \
  --preferred-challenges dns \
  --noninteractive \
  --agree-tos \
  --dns-azure-config $INI_FILE_PATH \
  -d "$DOMAIN"
```



# GCP Setup

## Install Plugin

```shell
pip3 install certbot-dns-google
```

## Service Account & Role Creation

Create a new service account and accompanying role. Parameters required:

* $PROJECT_ID - name of the project
* $KEY_FILE_PATH - path to a new output file for the private key—for example, ~/sa-private-key.json.

```shell
# create service account
gcloud iam service-accounts create certbot-dns-google \
  --project=$PROJECT_ID \
  --description="Certbot DNS" \
  --display-name="Certbot DNS Google"

# create role
gcloud iam roles create dns.certbot.admin \
  --project=$PROJECT_ID \
  --title="DNS Certbot Admin" \
  --description="Administers DNS Zones for Certbot" \
  --permissions="dns.changes.create,dns.changes.get,dns.changes.list,dns.managedZones.get,dns.managedZones.list,dns.resourceRecordSets.create,dns.resourceRecordSets.delete,dns.resourceRecordSets.list,dns.resourceRecordSets.update" \
  --stage=GA

# bind service account to role
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:certbot-dns-google@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="projects/$PROJECT_ID/roles/dns.certbot.admin"

# create service account key
gcloud iam service-accounts keys create $KEY_FILE_PATH \
  --iam-account=certbot-dns-google@$PROJECT_ID.iam.gserviceaccount.com
```

## Certificate Creation

Create your certificate. Parameters required:

* $KEY_FILE_PATH
* $DOMAIN

```shell
sudo certbot certonly \
  --dns-google \
  --dns-google-credentials $KEY_FILE_PATH \
  -d "$DOMAIN"
```