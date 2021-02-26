# How to integrate Zymbit with AWS IoT

## Introduction - Perhaps it's own article?
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


## Process Overview

### Global Setup
- Configure private certificate authority (optional)
  - Create private CA
  - Register CA with AWS
- Create an IAM role with GetRole and PassRole permissions
- Create a role trust policy for credentials provider to assume this role
- Create a role alias linked to IAM role
- Create an IoT policy which allows role alias to be assumed with a certificate

### Device Process
- Generate CSR with Zymkey
  - The CSR contains specific device info
- Sign CSR with private CA
- Put device cert and root CA cert onto device
- Register device cert in AWS with root CA cert
- Create a IoT Thing in AWS
- Attach thing to device cert
- Attach policy to device cert
- Curl credential provider url using TLS to receive AWS device credentials

## Global AWS Setup

### Prerequisites
This global setup can be done anywhere with AWS CLI v2 installed.

### Create a private certificate authority
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


### Register certificate authority with AWS
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
aws iot register-ca-certificate --ca-certificate CA_files/zk_ca.pem \
                                --verification-certificate verificationCert.csr \
                                --set-as-active \
                                --allow-auto-registration
```


### Create an IAM role with GetRole and PassRole permissions
1. Go into AWS IAM, create a role with role-pass-permissions.json and use the
given role trust policy.

Create role, click Certificate Manager, click Next,

### Create a role alias linked to IAM role

### Create an IoT policy which allows role alias to be assumed with a certificate


## Device Process
### Generate CSR with Zymkey
  - The CSR contains specific device info
### Sign CSR with private CA
### Put device cert and root CA cert onto device
### Register device cert in AWS with root CA cert
### Create a IoT Thing in AWS
### Attach thing to device cert
### Attach policy to device cert
### Curl credential provider url using TLS to receive AWS device credentials
