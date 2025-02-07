# Blog : Lambda function for Instance management

# Objective

Set up a system where an EC2 instance starts automatically when a Lambda function is triggered. Configure it so that when a new instance is created, its service automatically comes up. Additionally, ensure there is an option to terminate the instance using a Lambda function.

If developers are working on the server and want to save the state without terminating the instance directly, manage it by creating an AMI before termination. Additionally, provide an option to use the last AMI to resume the developers' work at server creation.

# Requirements

1. You should have an AWS account with atleast following permissions - 
    - AmazonEventBridgeFullAccess
    - AmazonDynamoDBFullAccess
    - iam:CreateServiceLinkedRole
    - iam:PutRolePolicy
    - iam:AttachRolePolicy
    - iam:PutRolePolicy
    - iam:AttachRolePolicy
    - iam:CreateServiceLinkedRole
    - ec2:*
    - scheduler:*

2. You should have your AWS Credentials like AWS Access key and AWS Secret Key along with region name.
3. You should have key pair installed in your system, if you would like to take the ssh access. Keep in mind to use correct ip. Here elastic ip is already assigned to instance.
4. Your Elastic ip should  be already created with name tag - **dev_elastic_ip**. In script **automatic_elastic_ip.sh**, we use this name tage to fetch the allocation id (not association id) then attach to EC2 instance in the same script.
5. Create a table in Dynamodb. Update all the following scripts with your new table name where you are going to store your configuration files. I had made the table `UserScripts` , just replace the name.

# Result

We created a Lambda function with four options: the first option creates a fresh EC2 instance with automatic configuration, the second option terminates the EC2 instance directly, the third option creates an EC2 instance using the last AMI and deletes that AMI, and the fourth option creates an AMI and then deletes the EC2 instance.

We created a Bash script to create a DynamoDB table and add data to it, as well as fetch data from the table. Additionally, we developed a batch script to encode and decode user data using Base64. Our collection of batch scripts is also useful for dockerizing an application with the provision of a deployment console

# Procedure

## Step 1: Create a lambda function to trigger the EC2 instance

This is the most important function which takes an event object. It generally contains the payload that transfers input to the function.

**Variables**

1. Input number: This number is generally provided by the event as a payload. 
2. Instance name: It represents the name tag that is assigned to the instance. 
3. Region: It represents a region where OK your instances are being created. 
4. AMI name: This is the AMI (name tag) which was created from the last scenario where instance was terminated, and data is backed up to create AMI. 
5. First AMI: It is the official AMI which is used to create the instance from scratch and then data is transferred to it. 
6. ec2_client: It is an ec2 client object that represents all the informations about EC2 service.
7. Functions Response: When we call another function within a Lambda function, it provides a response if an error occurs, making it useful for identifying the reason behind the error. create_instance_response, termination_response, change_state_response, create_ami_response, remove_ami_response, cancel_spot_response, first_instance_response represents the responses.

While starting the server, be sure that there should no instance running, it there is already an instance running with same name tag then it might exit the code graceful with a message. This logic is written in get_instance_id() function where we also provided an input variable.

```python
import boto3
import sys
import json
from datetime import datetime
import logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    AWS Lambda function to perform EC2 instance actions based on user input.
    """
    input_number = event.get('input_number')
    instance_name = "Dev_from_lambda"
    region = "your_region"
    ami_name = "DevServerImage-Latest"
    first_ami = "ami-053b12d3152c0cc71"
    ec2_client = boto3.client('ec2', region_name=region)

    try:
        logger.info(f"log - going to check instance availability")
        instance_id = get_instance_id(ec2_client, instance_name,input_number)
        logger.info(f"log - got instance id result as - {instance_id}")
    except Exception as e:
        logger.info(f"log - instance id fetching  - exceptional condition occoured ")
        return {
            'statusCode': 500,
            'body': f"Error: {str(e)}"
        } 
    
    create_instance_response = "None"
    termination_response = "None"
    change_state_response = "None"
    create_ami_response = "None"
    remove_ami_response = "None"
    cancel_spot_response = "None"
    first_instance_response = "None"
    
    if input_number == 1:
        logger.info(f"log - You selected to create an new instance with AMI specification")
        create_instance_response = create_instance(ec2_client,ami_name,instance_name)
        remove_ami_response = remove_ami(ec2_client,ami_name)
    elif input_number == 2:
        logger.info(f"log - You selected to terminate instance with its AMI's")
        create_ami_response = create_ami(ec2_client, instance_id,ami_name)
        termination_response = terminate_instance(ec2_client, instance_id)
        cancel_spot_response = cancel_spot_request(ec2_client)
    elif input_number == 3:
        logger.info(f"log - You selected to create only instance")
        first_instance_response = first_instance(ec2_client,first_ami,instance_name)
    elif input_number == 4:
        logger.info(f"log - You selected to terminate simple instance only")
        termination_response = terminate_instance(ec2_client, instance_id)
        cancel_spot_response = cancel_spot_request(ec2_client)
    elif input_number == 5:
        print("Five")
    else:
        print("Number is not between 1 and 5")
    
    try:
        response_data = {
            "statusCode": 200,
            "body": {
                "create_instance_response": create_instance_response,
                "termination_response": termination_response,
                "create_ami_response": create_ami_response,
                "remove_ami_response": remove_ami_response,
                "cancel_spot_response": cancel_spot_response,
                "first_instance_response" : first_instance_response
            }
        }
        return response_data
    except Exception as e:
        return {
            'statusCode': 500,
            'body': f"Error: {str(e)}"
        }

```

## Step 2: Set up the Payload for various operations

To instruct the Lambda function on the operation to perform, we provide user input through user data. This input typically contains a payload representing a number that indicates the specific action the Lambda function should execute. This payload is entered in Eventbridge.

```json
# first instance
{
  "input_number": 3
}

# Terminate instance only
{
  "input_number": 4
}

# start with AMI
{
  "input_number": 1
}

# Stop with AMI
{
  "input_number": 2
}

```

## Step 3: Get the permissions policy for Lambda function

We generally assign a role to the Lambda function, which includes a policy with all the necessary permissions, such as retrieving information from EC2 instances and generating logs in CloudWatch. We are also creating a spot instance that also needs some special permissions. 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "ec2:*",
                "iam:CreateServiceLinkedRole",
                "iam:PutRolePolicy",
                "iam:PassRole"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

## Step 4: Create an EventBridge Schedule for automatic triggering of Lambda function.

1. User the cron expression - 30 9 ? * MON-FRI * , with recurring schedule.
2. Set flexible window off.
3. Switch off the retry policy, if you face any issue, see cloudwatch logs for lambda function.
4. Give your lambda function.
5. Give a proper IAM role.
6. Give corresponding payload.

## Step 5: Test your Lambda function and see logs in cloudwatch

- **Go to CloudWatch Logs:**
    - You can open the **CloudWatch Console**: [CloudWatch Console](https://console.aws.amazon.com/cloudwatch/).
    - Click **Logs → Log Groups** in the left-hand menu.
- **Find Your Lambda Log Group:**
    - Lambda logs are stored under **`/aws/lambda/<function-name>`**.
    - Click on the log group for your function.
- **View Log Streams:**
    - Inside the log group, you'll find multiple **log streams** (each corresponding to a specific Lambda execution).
    - Click on the most recent log stream to see the execution logs.

# Theories

## Theory 1: Management codes

### 1. Management bash codes

We have the necessary permissions to create a table in DynamoDB, so we created a **UserScripts** table. If you want to create another table, you can simply change the table name and use the following code. 

To ensure that the correct user data is transferred to the instance, you can verify it using the specified path in the following code.

Automating server configuration requires multiple scripts. You can pass all the scripts to DynamoDB at once using the following code snippet. You should ensure that **add_data.sh** script should be already present in current directory with proper  executable permission and proper AWS CLI authenticated.

At the bottom of the following code snippet, you can see the code for putting data into DynamoDB. However, I strongly recommend not using this code directly. It is provided only for your understanding of how scripts are passed to DynamoDB. We generally use the `add_data.sh` script, which encrypts the data before sending it.

```bash

# create dynamo db table
aws dynamodb create-table \
    --table-name UserScripts \
    --attribute-definitions AttributeName=Name,AttributeType=S \
    --key-schema AttributeName=Name,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
    
# getting the user data
/var/lib/cloud/instances/INSTANCE_ID/user-data.txt

# uploading all the files from current directory
for file in *; do
  bash add_data.sh "$file"
done

nano file.sh
bash encry.sh file.sh

    
# put the data to table
variable=$(cat file.sh) 
aws dynamodb put-item \
    --table-name UserScripts \
    --item '{
        "Name": {"S": "Your-key"},
        "data": {"S": "'"$variable"'"} 
    }'
```

### 2. first_script.sh

The following code fetches all the configuration files and scripts from DynamoDB and executes them sequentially to ensure the server is set up and running properly. User data is the first code that executes when the instance starts for the first time. We generally use user data to set up the environment, ensuring it becomes ready to run the next script, such as `first_script.sh`.

```bash
#!/bin/bash

# fetching the scripts
bash fetch_data.sh add_data.sh
bash fetch_data.sh all_start.sh
bash fetch_data.sh all_stop.sh
bash fetch_data.sh generate_logs.sh
bash fetch_data.sh get_logs.sh
bash fetch_data.sh run_backend.sh
bash fetch_data.sh run_containers.sh
bash fetch_data.sh stop_containers.sh
bash fetch_data.sh stop_logging.sh
bash fetch_data.sh delete_image.sh
bash fetch_data.sh deploy.sh
bash fetch_data.sh pull_image.sh
bash fetch_data.sh show_containers.sh
bash fetch_data.sh show_images.sh
bash fetch_data.sh instructions.txt
bash fetch_data.sh backend.env
bash fetch_data.sh automatic_elastic_ip.sh
bash fetch_data.sh set_ssl_files.sh
bash fetch_data.sh start_all.service

# project directory
mkdir /application

# path for environment credentials
mkdir -p /application/Environment/backend

# path for scripts
mkdir /application/auto-scripts/

# path for building images
mkdir /application/build-scripts

# for deployemts
mkdir /application/deploy-scripts

# path for systemd files
mkdir /application/systemd-files

# setting the .env file
mv backend.env /application/Environment/backend/.env

# make the scripts executable
sudo chmod +x *

# setting sytemd
sudo ln -s /usr/local/bin/start_all.service /etc/systemd/system/
sudo systemctl daemon-reload

# pull the image
pull_image.sh

# start the server
systemctl start start_all.service

# setting ssl configuration
set_ssl_files.sh

# wait for instance to be running
sleep 180

# setting up elastic ip 
automatic_elastic_ip.sh
```

### 3. User data for Lambda function

Following function returns a text that represents user data, when called. When we create an EC2 instance using Lambda functions, we have the option to provide user data to the EC2 instance. In the user data, the first line should include the **shebang** (`#!/bin/bash` for bash scripts), as this tells the system which interpreter to use to execute the script. Without this, the script may not execute properly. 

In the following user data script, we created a file containing encrypted data. This file is then decrypted , which is a script that connects the server to DynamoDB, allowing it to fetch additional configuration files for further setup.

```python
def get_user_data():
    return '''#!/bin/bash
    cd ~
    echo "give-your-base-64-encrypted-user-data-with-removed-new-lines" > user_data.b64

    # Get the filename 
    filename="user_data.b64"

    # Extract the original filename (remove .b64 extension)
    original_filename="${filename%.*}"

    # Decode the Base64 file
    base64 -d < "$filename" > "$original_filename"

    mv user_data user_data.sh
    sudo chmod +x user_data.sh

    bash user_data.sh
    '''
```

### 4. data_before_encryption

We will encrypt the following script within the user data to make it more concise when passed to lambda function. You can use the scripts like [encry.sh](http://encry.sh) and [decry.sh](http://decry.sh) in order to encrypt it. Then use remove_line.sh to remove the line form encrypted file. Finally attach it with user data  in get_user_data() function. 

To ensure the following script works correctly, run it manually on your trial server. If you encounter any issues, ssh to your server, go to path `/var/lib/cloud/instances/INSTANCE_ID/user-data.txt` check the user script file by running it manually.

In order to configure the following script according to envirnment, give proper AWS Credentials and Dynamo db table name that you had created manually. Your table should already have `first_script.sh` along with dependent configuration files (required for first_script.sh)

You don't need to worry about permissions. The following script runs within the user data, which executes as the root user on the Linux system. But keep  in mind that providing sudo with privileged  commands is a good practice.

```bash
#!/bin/bash

sudo echo "User data script started" >> /tmp/user_data_log
# Installing Docker
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo echo "Docker installed" >> /tmp/user_data_log
sudo apt install -y nginx jq

sudo echo "Nginx and jq installed" >> /tmp/user_data_log

sudo snap install aws-cli --channel=v1/stable --classic

sudo echo "AWS CLI installed" >> /tmp/user_data_log

cd /usr/local/bin

aws configure set aws_access_key_id "your_aws_access_key"
aws configure set aws_secret_access_key "your_aws_secret_key"
aws configure set region "your_aws_region"

sudo echo "AWS CLI Setup done installed" >> /tmp/user_data_log

sudo echo "Going to get the fetch _data.sh script" >> /tmp/user_data_log

key_name="fetch_data.sh"
filename="${key_name}"

# Get the item from DynamoDB
item=$(aws dynamodb get-item \
    --table-name UserScripts \
    --key '{"Name": {"S": "'"$key_name"'"}}' \
    --output json | jq -r '.Item.data.S')

item=$(echo "$item" | base64 -d)
# Save the data to a file
echo "$item" > "$filename"
echo "Data saved to: $filename"

sudo chmod +x fetch_data.sh

sudo echo "We got fetch data script" >> /tmp/user_data_log

bash fetch_data.sh first_script.sh
sudo chmod +x first_script.sh

sudo echo "We got first script" >> /tmp/user_data_log

bash first_script.sh

```

### 5. add_data.sh

If any of your configuration files change, you can update them or add a new file in DynamoDB using the following script. Just change the table name. Your respected file will also encrypted before storing it into the table.

```bash
#!/bin/bash

if [ $# -eq 0 ]; then
  echo "Usage: $0 <filename>"
  exit 1
fi

filename="$1"

# Read the content of the file an encrypt it
file_content=$(cat "$filename" | base64)
file_content=$(echo "$file_content" | tr -d '\n')

# Construct the DynamoDB put-item command
aws dynamodb put-item \
    --table-name UserScripts \
    --item '{
        "Name": {"S": "'"${filename}"'"}, 
        "data": {"S": "'"$file_content"'"} 
    }'
```

### 6. fetch_data.sh

The following script fetches the configuration file from DynamoDB. You need to provide the proper file name as argument to this script.

```bash
#!/bin/bash 

# Function to fetch data from DynamoDB and save to a file, user argument as key to fetch the data
# Use same argument to create a new file (bash file)
fetch_and_save_data() {
  local key_name="$1"
  local filename="${key_name}"

  # Get the item from DynamoDB
  item=$(aws dynamodb get-item \
      --table-name UserScripts \
      --key '{"Name": {"S": "'"$key_name"'"}}' \
      --output json | jq -r '.Item.data.S')

  # Check if the item exists
  if [ -z "$item" ]; then
    echo "No data found for key: $key_name"
    return 1
  fi

  item=$(echo "$item" | base64 -d)
  # Save the data to a file
  echo "$item" > "$filename"
  echo "Data saved to: $filename"
}

# Check if at least one argument is provided
if [ $# -eq 0 ]; then
  echo "Error: Please provide a key name as an argument."
  exit 1
fi

# Get the key name from the first argument
key_name="$1"

# Call the function to fetch and save the data
fetch_and_save_data "$key_name"
```

### 7. start_all.service

The following file creates a daemon service that starts all other services, such as container creation and log initiation, without requiring a persistent terminal connection.

```bash
[Unit]
Description=Simple systemd service to start all the services
After=network.target

[Service]
ExecStart=all_start.sh
ExecStop=all_stop.sh
Restart=on-failure
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

### 8. decry.sh

If you have a Base64-encoded encrypted file, you can decrypt it using the following script.  This script can also be used to verify the integrity of the decrypted script.

```python
#!/bin/bash

# Check if a filename is provided as an argument
if [ -z "$1" ]; then
  echo "Usage: $0 <filename>"
  exit 1
fi

# Get the filename from the argument
filename="$1"

# Extract the original filename (remove .b64 extension)
original_filename="${filename%.*}"

# Decode the Base64 file
base64 -d < "$filename" > "$original_filename"

echo "File '$filename' decoded from Base64 and saved as '$original_filename'"

```

### 9. encry.sh

The following script encrypts a file. We will use this script to pass encrypted user data (data_before_encryption) to the Lambda function in the `get_user_data()` function.

```bash
#!/bin/bash

# Check if a filename is provided as an argument
if [ -z "$1" ]; then
  echo "Usage: $0 <filename>"
  exit 1
fi

# Get the filename from the argument
filename="$1"

# Create the output filename (appending .b64)
output_filename="${filename}.b64"

# Encode the file to Base64
base64 < "$filename" > "$output_filename"

echo "File '$filename' encoded to Base64 and saved as '$output_filename'"
```

### 10. remove_line.sh

Removing the new line from the encrypted data is crucial because its presence prevents the data from being included in the user data passed to the Lambda function. We can use following script to remove the new lines form encrypted data.

```bash
#!/bin/bash

if [ -z "$1" ]; then
  echo "Usage: $0 <filename>"
  exit 1
fi

filename="$1"

# Read the entire file content into a single variable
file_content=$(cat "$filename")

# Remove newline characters
file_content=$(echo "$file_content" | tr -d '\n')

# Write the modified content back to the file
echo "$file_content" > "$filename"

echo "Newline characters removed from '$filename'."

```

## Theory 2: Lambda Function

 

### 1.  Logging and imports

The following are the dependencies required for the Lambda function to operate.

```python
import boto3
import sys
import json
from datetime import datetime
import logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# user following line to log the output
logger.info(f"log - Logging message - {variable_at_righet_scopt} ")
```

### 2. cancel_spot_request

When creating and subsequently deleting a Spot Instance, it's crucial to also delete the associated Spot Instance Request.  This request is not automatically deleted, so manual deletion is required. The following function requires only the EC2 client object. It automatically retrieves all active Spot Instance Requests and deletes them.

```python

def cancel_spot_request(ec2_client):
    """
    Cancel spot request function
    """
    logger.info(f"log - Cancelling spot request started")
    try:
        ec2 = ec2_client
        
        # Fetch all spot instance requests
        spot_requests = ec2.describe_spot_instance_requests().get('SpotInstanceRequests', [])

        # Extract only SpotInstanceRequestId values
        spot_request_ids = [request['SpotInstanceRequestId'] for request in spot_requests]

        logging.info(f"log - getting the value of spot_request_ids - {spot_request_ids} ")

        if not spot_request_ids:
            return {
                'statusCode': 400,
                'body': 'No SpotRequestIds available.'
            }

        response = ec2.cancel_spot_instance_requests(
            SpotInstanceRequestIds=spot_request_ids
        )
        logger.info(f"log - Successfully canceled Spot Request(s): {spot_request_ids}")
        return f'Successfully canceled Spot Request(s): {spot_request_ids}'

    except Exception as e:
        print(f"Error canceling Spot Request: {e}")  # Log the error
        return {
            'statusCode': 500,
            'body': f'Error canceling Spot Request: {e}'
        }

```

### 3. get_snapshot_id

When an AMI is created, a snapshot is also generated. Deleting the AMI does *not* automatically delete the associated snapshot, we need to  delete it manually. We created a function that takes an EC2 client object and an AMI ID. It uses this information to retrieve the associated snapshot ID and returns it.

```python

def get_snapshot_id(ec2_client,ami_id):
    """
    Function that fetches corresponding snapshots from its associated ami_id
    """
    logger.info(f"log - Fetching snapshot id's ")
    try:
        # Describe the AMI to get the block device mapping
        response = ec2_client.describe_images(
            ImageIds=[ami_id]
        )
        
        # Check if AMI exists and has block device mappings
        if response['Images']:
            block_device_mappings = response['Images'][0].get('BlockDeviceMappings', [])
            
            # Extract Snapshot IDs from BlockDeviceMappings
            snapshot_ids = []
            for mapping in block_device_mappings:
                if 'Ebs' in mapping and 'SnapshotId' in mapping['Ebs']:
                    snapshot_ids.append(mapping['Ebs']['SnapshotId'])
            
            # If Snapshot IDs are found, return them
            if snapshot_ids:
                logger.info(f"log - Got snapshot id - {snapshot_ids} ")
                return snapshot_ids
            else:
                return "No snapshot IDs found for the AMI."
        else:
            return f"No AMI found with ID: {ami_id}"

    except Exception as e:
        return f"Error: {str(e)}"

```

### 4. get_ami_id

To create an EC2 instance from a custom AMI, the AMI ID is required. The following function retrieves the AMI ID from a provided AMI name. This function is called only when creating an instance from a custom AMI. If no AMI is found, the Lambda function exits gracefully with the message: "You tried to create a server with a custom AMI, but no images were found. Exiting program."

```python

def get_ami_id(ec2_client,ami_name):
    """
    Function that fetches corresponding AMI id from provided AMI name
    """
    logger.info(f"log - fetching ami id's from provided ami name {ami_name} ")
    # Create EC2 client
    ec2_client = boto3.client('ec2')

    try:
        # Describe images filtered by name
        response = ec2_client.describe_images(
            Filters=[{'Name': 'name', 'Values': [ami_name]}]
        )

        # Check if any images are returned
        if response['Images']:
            # Return the AMI ID of the first matched AMI
            ami_id = response['Images'][0]['ImageId']
            logger.info(f"log - got ami id - {ami_id} ")
            return ami_id
        else:
            # if images are not returned
            logger.info(f"log - you tried to create server with custom AMI, no images are found, exiting program")
            sys.exit(0)

    except Exception as e:
        return f"Error: {str(e)}"

```

### 5. create_ami

If developers working on a server want to preserve its state before termination, they can create a custom AMI. This is done before the instance is terminated.  The following function takes input as ec2 client object, instance ID and the desired name. 

```python

def create_ami(ec2_client, instance_id, ami_name):
    """
    Function that Creates an AMI from a given EC2 instance.
    """
    logger.info(f"log - AMI Creation started")
    try:
        timestamp = datetime.now().strftime("%Y%m%d%H%M%S")

        
        response = ec2_client.create_image(
            InstanceId=instance_id,
            Name=ami_name,
            Description=f"Custom AMI created from instance at {timestamp}",
            NoReboot=True  # Set to False if you want to reboot before AMI creation
        )
        
        ami_id = response['ImageId']
        logger.info(f"log - AMI {ami_id} is being created from instance {instance_id} ")
        return {
            'statusCode': 200,
            'body': json.dumps(f"AMI {ami_id} is being created from instance {instance_id}.")
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps(f"Error: {str(e)}")
        }

```

### 6. remove_ami

When a custom AMI have been used to restore an instance, the AMI is no longer needed.  It's good idea to delete the custom AMI after the server is up and running to avoid unnecessary storage costs. Following function takes EC2 client object and custom AMI name and input number.  AMI name will be helpful to fetch the AMI id and delete it.

```python

def remove_ami(ec2_client,ami_name):
    """
    Function that removes Custom AMI if that AMI is in use of running container
    """
    logger.info(f"log - Image removing process started")
    try:
        ami_id = get_ami_id(ec2_client,ami_name)
        snapshot_ids = get_snapshot_id(ec2_client,ami_id)
        
        # Deregister the AMI
        ec2_client.deregister_image(ImageId=ami_id)
        logger.info(f"log - AMI {ami_id} deregistered successfully.")
        
        # Delete associated snapshots
        for snapshot_id in snapshot_ids:
            ec2_client.delete_snapshot(SnapshotId=snapshot_id)
            logger.info(f"Snapshot {snapshot_id} deleted successfully.")
        
        return {
            'statusCode': 200,
            'body': f"AMI {ami_name} with ID {ami_id} and associated snapshots deleted successfully."
        }
    
    except Exception as e:
        logging.error(f"Error: {e}")
        return {
            'statusCode': 500,
            'body': f"An error occurred: {e}"
        }

```

### 7. first_instance

This function is used when creating an EC2 instance from scratch, without using a custom AMI.  The server will be configured using configuration files. The input taken by this function are ec2 client object, AMI  id (AWS official AMI id) and your desired instance name. Inside this function, we also calls get_user_data() function to provide the custom user data, which will be helpful for setting up the server.

```python

def first_instance(ec2_client,first_ami,instance_name):
    """
    Function that Start an EC2 spot instance from scratch with provided user data
    """
    logger.info(f"log - First Instance Creation started")

    ami_id = first_ami
    try:
        params = {
            "ImageId": ami_id,
            "InstanceType": "t2.micro",
            "KeyName": "my_ssh_key",
            "SecurityGroupIds": ["sg-some-id"],
            "SubnetId": "subnet-some-id",
            "MinCount": 1,
            "MaxCount": 1,
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/sda1",
                    "Ebs": {
                        "VolumeSize": 10,
                        "VolumeType": "gp3"
                    }
                }
            ],
            "TagSpecifications": [
                {
                    'ResourceType': 'instance',
                    'Tags': [
                        {'Key': 'Name', 'Value': instance_name}
                    ]
                }
            ],
            "InstanceMarketOptions": {
                "MarketType": "spot",
                "SpotOptions": {
                    "MaxPrice": "0.006",
                    "SpotInstanceType": "one-time",
                    "InstanceInterruptionBehavior": "terminate"
                }
            },
            "UserData": get_user_data() 
        }
        
        instances = ec2_client.run_instances(**params)
        instance_id = instances['Instances'][0]['InstanceId']
        
        logger.info(f"log - Created First EC2 instance with ID: {instance_id}")
        
        return {
            'statusCode': 200,
            'body': f"First EC2 instance {instance_id} created successfully."
        }
    except Exception as e:
        print(f"Error creating EC2 instance: {str(e)}")
        return {
            'statusCode': 500,
            'body': f"Failed to create First EC2 instance: {str(e)}"
        }

```

### 8. create_instance

This function creates an EC2 instance using a custom AMI, which is used to restore the server to a previous state. This function takes 3 inputs, namely ec2 client object, AMI name which contains the state of previous EC2 instance and instance name to name tag.

```python

def create_instance(ec2_client,ami_name,instance_name):
    """
    Starting an EC2 spot instance with using avaible AMI 
    """
    logger.info(f"log - Instance Creation from Custom ami - {ami_name} , started")

    ami_id = get_ami_id(ec2_client,ami_name)

    try:
        params = {
            "ImageId": ami_id,
            "InstanceType": "t2.micro",
            "KeyName": "my_ssh_key",
            "SecurityGroupIds": ["sg-some-id"],
            "SubnetId": "subnet-some-id",
            "MinCount": 1,
            "MaxCount": 1,
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/sda1",
                    "Ebs": {
                        "VolumeSize": 10,
                        "VolumeType": "gp3"
                    }
                }
            ],
            "TagSpecifications": [
                {
                    'ResourceType': 'instance',
                    'Tags': [
                        {'Key': 'Name', 'Value': instance_name}
                    ]
                }
            ],
            "InstanceMarketOptions": {
                "MarketType": "spot",
                "SpotOptions": {
                    "MaxPrice": "0.006",
                    "SpotInstanceType": "one-time",
                    "InstanceInterruptionBehavior": "terminate"
                }
            },
            "UserData": """#!/bin/bash
            cd /usr/local/bin
            bash fetch_data.sh automatic_elastic_ip.sh
            bash automatic_elastic_ip.sh
            """
        }
        
        instances = ec2_client.run_instances(**params)
        instance_id = instances['Instances'][0]['InstanceId']
        
        logger.info(f"log - Created EC2 instance with ID: {instance_id}")
        
        return {
            'statusCode': 200,
            'body': f"EC2 instance {instance_id} created successfully."
        }
    except Exception as e:
        print(f"Error creating EC2 instance: {str(e)}")
        return {
            'statusCode': 500,
            'body': f"Failed to create EC2 instance: {str(e)}"
        }

```

### 9. get_instance_id

As name suggest, it returns the instance id with provide instance name. It is a crucial function as it checks for existing running instances with the same name tag. If a running instance is found when a request for a new instance with the same name is received, the new instance request is rejected with a message indicating the conflict. This function takes inputs like ec2 client object, instance name and input number.

```python

def get_instance_id(ec2_client, instance_name,input_number):
    """
    This function Fetches the instance ID based on the Name tag.
    """
    logger.info(f"log - Fetching instance id on the base of instance name - {instance_name}")
    try:
        response = ec2_client.describe_instances(
            Filters=[
                {'Name': 'tag:Name', 'Values': [instance_name]},
                {'Name': 'instance-state-name', 'Values': ['running']}  # Filter for running instances
            ]
        )

        instances = response.get('Reservations', [])
        if instances and instances[0]['Instances']:
            # fount an instance id
            logger.info(f"log - got the instance id - id is {instances[0]['Instances'][0]['InstanceId']}")
            if input_number == 2 or input_number == 4:
                return instances[0]['Instances'][0]['InstanceId']  # Return first running instance ID
        else:
            # not found an instance id
            logger.info(f"log - no instance found")
            if input_number == 3 or input_number == 1:
                logger.info(f"log - You are going to create an instance")
                return "No Instance found, You are creating instance with fresh image, from next line"
        logger.info(f"log - you tried to delete instance, while no instance is running, exiting program gracefully")
        sys.exit(0)
    except Exception as e:
        return {
            'statusCode': 500,
            'body': f"Error: {str(e)}"
        }

```

### 10. terminate_instance

```python

def terminate_instance(ec2_client, instance_id):
    """
    Instance Terminates function to terminate an EC2 instance given its Instance ID.
    """
    logger.info(f"log - Insance Termination started")
    try:
        response = ec2_client.terminate_instances(InstanceIds=[instance_id])  
        logger.info(f"log - Terminated EC2 Instance: {instance_id}")      
        return {
            'statusCode': 200,
            'body': f"Terminating EC2 Instance: {instance_id}"
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': f"Error: {str(e)}"
        }

```

### 11. get_user_data

```python
def get_user_data():
    return '''#!/bin/bash
    cd ~
    echo "your-encrypted-user-data" > user_data.b64

    # Get the filename 
    filename="user_data.b64"

    # Extract the original filename (remove .b64 extension)
    original_filename="${filename%.*}"

    # Decode the Base64 file
    base64 -d < "$filename" > "$original_filename"

    mv user_data user_data.sh
    sudo chmod +x user_data.sh

    bash user_data.sh
    '''
```

## Theory 3: Securing The server

When creating a server, security is important, making SSL connection setup essential. This can be achieved by installing Nginx on the Ubuntu server and placing the certificate in the appropriate directory.  Proper configuration of the Nginx configuration file is also necessary. The following script accomplishes the same. Ensure that all necessary files are already present in your DynamoDB table with proper table name.

### set_ssl_files.sh

```bash
#!/bin/bash

fetch_data.sh accounts-acme-v02.api.letsencrypt.org-directory-5a3e6a5f02752f3ef2f7b8a7359664af-meta.json  
fetch_data.sh accounts-acme-v02.api.letsencrypt.org-directory-5a3e6a5f02752f3ef2f7b8a7359664af-private_key.json  
fetch_data.sh accounts-acme-v02.api.letsencrypt.org-directory-5a3e6a5f02752f3ef2f7b8a7359664af-regr.json  
fetch_data.sh archive-devapi.itshobhit.co.in-cert1.pem  
fetch_data.sh archive-devapi.itshobhit.co.in-chain1.pem  
fetch_data.sh archive-devapi.itshobhit.co.in-fullchain1.pem  
fetch_data.sh archive-devapi.itshobhit.co.in-privkey1.pem  
fetch_data.sh live-README  
fetch_data.sh live-devapi.itshobhit.co.in-README  
fetch_data.sh options-ssl-nginx.conf  
fetch_data.sh ssl-dhparams.pem  
fetch_data.sh renewal-devapi.itshobhit.co.in.conf  
fetch_data.sh updated-options-ssl-nginx-conf-digest.txt
fetch_data.sh updated-ssl-dhparams-pem-digest.txt
fetch_data.sh set_ssl_permission.sh
fetch_data.sh application-config

# create directories

# Define base directory (change if needed)
BASE_DIR="/etc/letsencrypt"

# Create directory structure
mkdir -p "$BASE_DIR"
mkdir -p "$BASE_DIR/accounts/acme-v02.api.letsencrypt.org/directory/5a3e6a5f02752f3ef2f7b8a7359664af"
mkdir -p "$BASE_DIR/archive/devapi.itshobhit.co.in"
mkdir -p "$BASE_DIR/live/devapi.itshobhit.co.in"
mkdir -p "$BASE_DIR/renewal"
mkdir -p "$BASE_DIR/renewal-hooks/deploy"
mkdir -p "$BASE_DIR/renewal-hooks/post"
mkdir -p "$BASE_DIR/renewal-hooks/pre"

# set up json files

mv  accounts-acme-v02.api.letsencrypt.org-directory-5a3e6a5f02752f3ef2f7b8a7359664af-meta.json  "$BASE_DIR/accounts/acme-v02.api.letsencrypt.org/directory/5a3e6a5f02752f3ef2f7b8a7359664af/meta.json"
mv  accounts-acme-v02.api.letsencrypt.org-directory-5a3e6a5f02752f3ef2f7b8a7359664af-private_key.json  "$BASE_DIR/accounts/acme-v02.api.letsencrypt.org/directory/5a3e6a5f02752f3ef2f7b8a7359664af/private_key.json"
mv  accounts-acme-v02.api.letsencrypt.org-directory-5a3e6a5f02752f3ef2f7b8a7359664af-regr.json  "$BASE_DIR/accounts/acme-v02.api.letsencrypt.org/directory/5a3e6a5f02752f3ef2f7b8a7359664af/regr.json"

# Set up README files
mv live-README "$BASE_DIR/live/README"
mv live-devapi.itshobhit.co.in-README "$BASE_DIR/live/devapi.itshobhit.co.in/README"

# Set up config file
mv renewal-devapi.itshobhit.co.in.conf "$BASE_DIR/renewal/devapi.itshobhit.co.in.conf"

# Set up SSL options and parameters files
mv options-ssl-nginx.conf "$BASE_DIR/options-ssl-nginx.conf"
mv ssl-dhparams.pem "$BASE_DIR/ssl-dhparams.pem"

# Set up certificates file
mv archive-devapi.itshobhit.co.in-cert1.pem "$BASE_DIR/archive/devapi.itshobhit.co.in/cert1.pem"
mv archive-devapi.itshobhit.co.in-chain1.pem "$BASE_DIR/archive/devapi.itshobhit.co.in/chain1.pem"
mv archive-devapi.itshobhit.co.in-fullchain1.pem "$BASE_DIR/archive/devapi.itshobhit.co.in/fullchain1.pem"
mv archive-devapi.itshobhit.co.in-privkey1.pem "$BASE_DIR/archive/devapi.itshobhit.co.in/privkey1.pem"

# set hidden files
mv updated-options-ssl-nginx-conf-digest.txt "$BASE_DIR/.updated-options-ssl-nginx-conf-digest.txt"
mv updated-ssl-dhparams-pem-digest.txt "$BASE_DIR/.updated-ssl-dhparams-pem-digest.txt"

# Create symbolic links for SSL certificates
ln -s "$BASE_DIR/archive/devapi.itshobhit.co.in/cert1.pem" "$BASE_DIR/live/devapi.itshobhit.co.in/cert.pem"
ln -s "$BASE_DIR/archive/devapi.itshobhit.co.in/chain1.pem" "$BASE_DIR/live/devapi.itshobhit.co.in/chain.pem"
ln -s "$BASE_DIR/archive/devapi.itshobhit.co.in/fullchain1.pem" "$BASE_DIR/live/devapi.itshobhit.co.in/fullchain.pem"
ln -s "$BASE_DIR/archive/devapi.itshobhit.co.in/privkey1.pem" "$BASE_DIR/live/devapi.itshobhit.co.in/privkey.pem"

# setting config file and restart nginx
mv application-config /etc/nginx/sites-available/application
rm -rf /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/application /etc/nginx/sites-enabled/application
sudo systemctl restart nginx
```

### automatic_elastic_ip.sh

SSL can only be set up if the IP address and domain name are same when a new server is created. While a server can we can retain the configuration for a given domain name by using the configuration file but its public IP address gets changed. This issue can be resolved by using an Elastic IP and associating it with the newly created server. The following script retrieves the instance ID from a provided instance name and the allocation ID from a provided Elastic IP name. After associating the Elastic IP with the corresponding instance, all requirements for SSL setup are met.

```bash
#!/bin/bash

# fetching instance id 
INSTANCE_NAME="Dev_from_lambda"
EIP_NAME="dev_elastic_ip"

INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=$INSTANCE_NAME" "Name=instance-state-name,Values=running" \
    --query "Reservations[*].Instances[*].InstanceId" \
    --output text)

# fetching allocation id of elastic ip
ALLOCATION_ID=$(aws ec2 describe-addresses \
    --filters "Name=tag:Name,Values=$EIP_NAME" \
    --query "Addresses[*].AllocationId" \
    --output text)

# Assign Elastic IP to EC2 Instance
aws ec2 associate-address --instance-id "$INSTANCE_ID" --allocation-id "$ALLOCATION_ID"

# Check if the command was successful
if [ $? -eq 0 ]; then
    echo "Elastic IP assigned successfully to instance $INSTANCE_ID"
else
    echo "Failed to assign Elastic IP. Check your AWS credentials and permissions."
fi

```
