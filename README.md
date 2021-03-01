# How to integrate Zymbit with AWS IoT

## Introduction
It is impossible to guarantee the successful operation of a remote device
indefinitely. The number of ways a device can be attacked may be infinite.
The highest security standard then must assume remote devices can be penetrated
at any time and plan for their failure.

From this, it is crucial to have the ability to identify and silence
specific devices from a fleet. This requires two parts:
1. Provide a unique identity for each device.
2. Manage the allowed operations of each device.

A unique device ID allows an administrator to specify what device to target.
An ID can be generated by a hash of serial numbers and other unique
device identifiers.

To manage device operations requires a central server to store and index device
certificates, policies and attributes. Luckily, AWS IoT provides these features.

Using AWS IoT also allows device data to be used by other AWS cloud services.
AWS services requires each IoT device must have valid credentials issued by AWS.


# Process Overview

## Global Setup
- Configure private certificate authority (optional)
  - Create private CA
  - Register CA with AWS
- Create an IAM role with GetRole and PassRole permissions
- Create a role trust policy for credentials provider to assume this role
- Create a role alias linked to IAM role
- Create an IoT policy which allows role alias to be assumed with a certificate

## Device Process
- Generate CSR with Zymkey
  - The CSR contains specific device info
- Sign CSR with private CA
- Put device cert and root CA cert onto device
- Register device cert in AWS with root CA cert
- Create a IoT Thing in AWS
- Attach thing to device cert
- Attach policy to device cert
- Curl credential provider url using TLS to receive AWS device credentials

# Global AWS Setup

## Prerequisites
This global setup can be done anywhere with the AWS CLI installed and an associated user with the AWS CLI.

#### Quick tutorial of how to setup a user to use AWS CLI.
 - Go to AWS IAM console, click users, add user, input a user name, and check programatic access.
 - Click Next:Permissions, "Attach existing policies directly", click AdministratorAccess
 - Click Next:Tags, Next:Review, Create user. Stay on this page.
 - On your device, run `aws configure` and fill in the appropriate values from the AWS page.

## Create a private certificate authority
On the device you want to hold your private CA and sign requests, do the following.

**Copy the lines below into a script called mk_ca.sh.**
```
#!/bin/bash
set -e
mkdir CA_files
cd CA_files

openssl ecparam -genkey -name prime256v1 -out zk_ca.key
OPENSSL_CONF=/etc/ssl/openssl.cnf openssl req \
  -x509 -new -SHA256 -nodes -key zk_ca.key \
  -days 3650 -out zk_ca.crt \
  -subj "/C=US/ST=California/L=Santa Barbara/O=Zymkey/CN=zymkey-verify.zymbit.com.dev"

cp zk_ca.crt zk_ca.pem
```
**Then run the script with the following command:**
```
bash mk_ca.sh
```

There are now three file in the directory CA_files.
  1. zk_ca.key
    - The private key for the CA, used for signing CSR's
  2. zk_ca.pem
    - The certificate for the CA in PEM format
  3. zk_ca.crt
    - The certificate for the CA


## Register certificate authority with AWS
On your private CA run the following commands.
1. Copy the registration code for generating CA cert.
```
aws iot get-registration-code
```

2. Create a private key for AWS CA to verify against.
```
openssl genrsa -out verificationCert.key 2048
```

3. Create a CSR for your CA to sign
```
openssl req -new -key verificationCert.key -out verificationCert.csr
```

4. Put registration code in the Common Name field
```
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []: 9c9df696a8a09688d30e9040b4b719129e4d6dbd6a898eeb0c654af0a5753b41
Email Address []:
```

5. Create a private key verification certificate for your CA. If you didn't follow
our CA creation section, then the -CA and -CAkey file paths are likely different.
```
openssl x509 -req -in verificationCert.csr -CA CA_files/zk_ca.pem \
-CAkey CA_files/zk_ca.key -CAcreateserial -out verificationCert.crt \
-days 500 -sha256
```

6. Register CA certificate, set it as active and allow device certificates to
auto register.
```
aws iot register-ca-certificate --ca-certificate file://CA_files/zk_ca.crt \
                                --verification-certificate file://verificationCert.crt \
                                --set-as-active \
                                --allow-auto-registration
```

## Create an IAM role for credentials provider
```
aws iam create-role --role-name credential_helper --assume-role-policy-document file://aws-integration/role-trust-policy.json
```

## Create GetRole/PassRole policy
In user-pass-permissions.json, substitute ACCOUNT_ID, with the "Account" field from 
```
aws sts get-caller-identity
```
Under ROLE_NAME, substitute the role name you created in the previous step. Then run the following command,
```
aws iam create-policy --policy-name <NAME> \
                      --policy-document file://aws-integration/user-pass-permissions.json
```
Copy the policy ARN.

## Create a user
```
aws iam create-user --user-name <NAME> \
                    --permissions-boundary <Policy ARN>
```


## Create a role alias linked to role
If you need your roleARN you can run `aws iam get-role --role-name <NAME>`
```
aws iot create-role-alias \
       --role-alias deviceRoleAlias \
       --role-arn <roleARN>
```


## Create an IoT policy which allows role alias to be assumed with a certificate
In iot-role-policy.json, substitue REGION with the desired region, ACCOUNT_ID with the ID previously found and ALIAS with the role alias name. Then run the following command,
```
aws iot create-policy \
    --policy-name credentialHelper \
    --policy-document file://aws-integration/iot-role-policy.json
```
Record the policyARN.

# Device Process - provision-device.sh
  - Generate CSR with Zymkey
    - The CSR contains specific device info
  - Sign CSR with private CA
  - Put device cert and root CA cert onto device
  - Register device cert in AWS with root CA cert
  - Create a IoT Thing in AWS
  - Attach thing to device cert
  - Attach policy to device cert
  - Curl credential provider url using TLS to receive AWS device credentials
