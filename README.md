# client-vpn

[CloudFormation](https://aws.amazon.com/cloudformation/) template for setting up a [Client VPN](https://aws.amazon.com/vpn/) endpoint to securely access private resources in a [VPC](https://aws.amazon.com/vpc/).

## Prepare

Make sure to have [AWS CLI](https://aws.amazon.com/cli/) installed and configure on your workstation.

Firstly, you need a self-signed certificate for encrypted tunnels.
Use below commands to generate a key-pair and import it into [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/):

```shell
# download easy-rsa
git clone https://github.com/OpenVPN/easy-rsa.git ~/easyrsa
cd ~/easyrsa/easyrsa3

# generate server certificates
./easyrsa build-ca nopass
./easyrsa build-server-full server nopass

# import the certificate
aws acm import-certificate \
  --certificate file://pki/issued/server.crt \
  --private-key file://private/server.key \
  --certificate-chain file://pki/ca.crt
```

Take note of the ARN for the imported certificate.

1. Now go to [Identity Center](https://aws.amazon.com/iam/identity-center/) in **AWS Console**.
2. Create a new **Custom SAML 2.0 application**, name it e.g., `VPN`.
3. In the **Application metadata** fields, use below values:
  - Application ACS URL: `http://127.0.0.1:35001`
  - Application SAML audience: `urn:amazon:webservices:clientvpn`
4. Download the **metadata file** from the creation page.
5. Click on **Edit attributes mapping** from the **Actions** menu for your app and and configure the mappings shown below.

| User attribute in the application | Maps to this string value or user attribute | Format         |
| --------------------------------- | ------------------------------------------- | -------------- |
| **Subject**                       | `${user:email}`                             | `emailAddress` |
| **Name**                          | `${user:email}`                             | `unspecified`  |
| **FirstName**                     | `${user:givenName}`                         | `unspecified`  |
| **LastName**                      | `${user:familyName}`                        | `unspecified`  |
| **memberOf**                      | `${user:groups}`                            | `unspecified`  |

6. Once created, you might want to assign some users to your new app.
7. Now go to [IAM](https://aws.amazon.com/iam/) in **AWS Console**.
8. Go to **Identity providers** and click on **Add provider** button.
9. Enter a **Provider name** e.g., `VPN` and choose the **metadata file** downloaded previously.
10. Click on **Add provider** and take note the or **ARN** for newly added provider.

## Deployment

In project folder, run below commands to deploy the stack:

```shell
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --stack-name client-vpn \
  --template-file template.yml \
  --parameter-overrides \
  "SamlProviderArn=???" \ # replace this
  "ServerCertificateArn=???" \ # replace this
  "SubnetId=???" \ # replace this
  "VpcId=???" # replace this
```

Once created, our new [Client VPN](https://aws.amazon.com/vpn/) setup is ready for use. 

```shell
# get the client VPN endpoint ID
aws cloudformation describe-stacks \
  --stack-name client-vpn \
  --query "Stacks[0].Outputs[?OutputKey=='VpnEndpointId'].OutputValue" \
  --output text

# get the SSP URL
aws ec2 describe-client-vpn-endpoints \
  --client-vpn-endpoint-ids ??? \ # replace this
  --query "ClientVpnEndpoints[0].SelfServicePortalUrl" \
  --output text
```

You can now download the [AWS Client VPN](https://aws.amazon.com/vpn/client-vpn-download/) and related config from the self-service portal URL obtained above.

## Extras

The stack also exports a the [Security Group](https://docs.aws.amazon.com/managedservices/latest/userguide/about-security-groups.html) ID associated with created [Client VPN](https://aws.amazon.com/vpn/) endpoint. You can use this value in your other [CloudFormation](https://aws.amazon.com/cloudformation/) templates to restrict access to certain resources.

```yml
AWSTemplateFormatVersion: 2010-09-09
Description: Example CloudFormation template.

Resources:
  ExampleSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: ExampleSecurityGroup
      GroupDescription: Example security group.
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          # use it like below
          SourceSecurityGroupId: !ImportValue client-vpn-VpnSecurityGroupId
```

## Credits

Coded with ❤️ love from 🇮🇳 India for everyone by [Syncloud Softech](https://syncloudsoftech.com/).
