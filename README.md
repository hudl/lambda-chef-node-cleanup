# Automatically Delete Terminated Instances in Chef Server with AWS Lambda
Using CloudWatch Events, when an instance is terminated a Lambda function is triggered that will remove the node from Chef server for you.  For this we'll use Lambda, CloudWatch Events, and AWS KMS.

**WARNING:  This code is meant as reference material only.  Using this code may cost you money.  Please be sure you understand your current usage and the costs associated with this reference code before launching in your AWS account.**

## Details
When an instance terminates, CloudWatch events will pass a JSON object containing the Instance ID to a Lambda function.  The JSON object does not contain any other identifying information of the instance, such as DNS name or Public IP Address.  Additionally, since the instance is now in a terminated state we cannot query any other identifying information about the instance.  This is important to understand because it effects how we must query for the node in Chef Server in order to delete it automatically.

The Lambda function then communicates with the Chef Server using a request hashed with a valid private key of a valid Chef Server user with appropriate permissions.  The Lambda expects an AWS KMS encrypted version of the private key which it will decrypt on the fly to sign all requests to the Chef Server.  The Lambda then makes a request to find a matching node in the Chef Server and finally a request to delete that node.

# Prerequisites
## Terraform
If you'd like to quickly deploy the reference, install [Terraform](https://www.terraform.io) which will help setup required components.  If you already have the [AWS CLI tools](https://aws.amazon.com/cli/) installed, with a credential profile setup, no further action is required.

If you do not have the AWS CLI tools installed, or any other AWS SDK, you should consider creating a [credential profile](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-config-files) file.  Otherwise, Terraform will prompt you to enter the Access Key and Secret Key for a user with permissions able to provision resources (IAM Role, Lambda, and CloudWatch Event).
## Deploying the Lambda Function
The included Terraform configuration files will create a Lambda function using a zip file named `lambda_function_payload.zip` in the parent directory (already present in this repository).  The uncompressed function and required dependencies can be found in the `lambda` directory.  Updating the zip and running `terraform apply terraform` from the parent directory will create a new version of the Lambda.
## KMS
Chef Server uses public key encryption to authenticate API requests.  This requires the client to hash the requests using a valid private key.  With this example, we'll use KMS to store an encrypted copy of our private key and then decrypt it on the fly with the Lambda function.

1. [Create a customer master key (CMK) in KMS](http://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html) and note the keyId that is automatically generated.
  * If you will use the supplied Terraform example in this repository you do not need to add a Key User yet.  If you are following this as a reference and already have an IAM role for your Lambda function you can add it now as a Key User.  Your IAM user needs kms.encrypt permissions to encrypt the certificate, while your Lambda user (via an IAM role) needs kms.decrypt permissions at runtime to access the certificate.
2. Encrypt the certificate in KMS using the AWS CLI tools:  `aws kms encrypt --key-id KEY_FROM_STEP_1 --plaintext file://your_private_key.pem`
  * aws kms encrypt --key-id 36c03bba-c32d-4162-8a12-10037963ffbc --plaintext fileb://chef1x.pem --query CiphertextBlob --output text|base64 -D|base64 -b 76 > chef1x_encrypted.pem (on osx)
3.	You will receive a response with a CiphertextBlob if successful.  An example of a successful response will look like:
  ```
  {
      "KeyId": "arn:aws:kms:us-east-1:123456789000:key/14d2aba8-5142-4612-a836-7cf17284c8fd",
      "CiphertextBlob": "CiCgJ6/K9CIXrDdsJ1fES7kBIJ0STEn+VwpMBjzsHVnH2xKQAQEBAgB4oCevyvQiF6w3bCdXxEu5ASCdEkxJ/lcKTAY87B1Zx9sAAABnMGUGCSqGSIb3DQEHBqBYMFYCAQAwUQYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAyk4nsWzRAWTiU4syoCARCAJDHOtYNdSYI6wlso8SgATXKJ0WF5s3qhLcVqMKxaTOO3bCI6Lw=="
  }
  ```

4. Copy this CiphertextBlob into a new file and store it in the same directory as the Lambda function; this is required so it can be packaged up with the function itself. I’ve used encrypted_pem.txt as the file name in my example, given the encrypted object is a certificate and private key, which is commonly name with the .pem file extension. Note the CiphertextBlob output is base64 encoded by the AWS CLI unless you send the output to a binary file using the fileb:// parameter. See the AWS KMS CLI help for more information on input and output encoding.

## Lambda Function
Modify the `REGION`, `CHEF_SERVER_URL`, and `USERNAME` variables as appropriate in `lambda/main.py`.

## Chef Server Permissions
The user making the request needs the appropriate permissions in Chef Server to query and delete nodes.  As described above, you'll need access to the private key for this user.

## Chef Nodes
The Lambda function is expecting that all nodes/instances managed by Chef have an attribute called `ec2_instance_id` with a value of the EC2 Instance ID (e.g. i-abcde123).  If this attribute is not present or not populated properly the function will not delete the node.

### Some Alternatives to the ec2_instance_id Attribute
Remember, CloudWatch Events will give us the Instance ID when an instance is terminated, but at that point other distinguishing information, like FQDN and IP Address, are already gone.  Using the Instance ID as an attribute or as a node name are probably the most convenient options, but they are not the only options.

#### Naming Nodes with the Instance ID
Instead of using an attribute, a simple alternative would be to name all nodes using their Instance ID.  Then you can modify the Lambda function to just fetch the Node by name instead of using "Search".
```
node = Node('instance-id')
node.delete()
```

#### AWS Config
If you prefer to not explicitly name your nodes and you do not want to include an attribute, another option is to let Chef use the fully qualified domain name (FQDN) of the instance as the node name (I believe this is the default behavior if you don't assign a name to a node during bootstrapping).  You can then query AWS Config in the Lambda function to retrieve the PrivateDNSName attribute and reconstruct the FQDN as it is a known pattern (ip-172-31-23-14.us-west-2.compute.internal).  Like above, you can then simply change the Lambda function to query for the node directly.

# Using This Example
Assuming the above Prerequisites section was followed and you've modified the Lambda function to include your Chef Server information, the next step is to build the payload.  From the parent directory you can run `zip -r lambda_function_payload.zip lambda/*` which creates a zip file containing everything AWS Lambda needs to run the code.  The name `lambda_function_payload.zip` is what the Terraform configuration expects the file to be named, so if you change it be sure to update `terraform/main.tf` as well.

Then, simply run `terraform apply terraform` from the parent directory.  This will create all the infrastructure resources required and deploy the Lambda function in the us-west-2 region.  To change the region, modify the `region` variable in `lambda/variables.tf`.

After running Terraform, you will need to manually add the IAM Role created as a Key User for the KMS Key you created earlier.  You can do this by using the console and adding the role name that was printed to the screen as output from Terraform ("chef_node_cleanup_lambda", by default).

## If you don't want to use Terraform
If you'd prefer to not use Terraform, you should still follow the Prerequisites section to get setup.  Then you'll need to do the following manually:

1. Create an IAM Role and Policy for the Lambda function.  Optionally use the builtin "lambda_basic_execution" IAM role.
* Upload the Lambda function.
* Setup a CloudWatch Event to invoke the Lambda function.

# I Just Want The Lambda Function!
The Lambda function code can be found at `lambda/main.py` for your reference.  Everything within the `lambda/` directory of this repository makes up the required files needed to run the Lambda as is so be sure to zip it all up.  You'll still want to consult the Prerequisites section to understand a few things, though.

# Destroying
If you used Terraform, you can cleanup with `terraform destroy terraform`

# License
Copyright 2016 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at

    http://aws.amazon.com/apache2.0/

or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

Note: Other license terms may apply to certain, identified software files contained within or distributed with the accompanying software if such terms are included in the directory containing the accompanying software. Such other license terms will then apply in lieu of the terms of the software license above.
