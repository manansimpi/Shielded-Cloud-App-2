# Shielded-Cloud-App-2
This is a sequel, so please start from the first part.

## 🟠 Phase 3: Setting up AMI's and Auto-Scaling Groups.

* Note. Before we start working on AMI's we need to make changes on our APP-Server so that we maintain the same configurations in the new upcoming Servers that would be automatically spinning up by ASG's.

1. Create a new script file:
```  sudo nano /usr/local/bin/update_metadata.sh  ```
 
- Add the following content:
  
 ```
#!/bin/bash
# Get instance metadata using IMDSv2
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
PUBLIC_IP=$(curl -sH "X-aws-ec2-metadata-token: $TOKEN" "http://169.254.169.254/latest/meta-data/public-ipv4")
HOSTNAME=$(curl -sH "X-aws-ec2-metadata-token: $TOKEN" "http://169.254.169.254/latest/meta-data/hostname")

# Validate metadata response
if [[ "$PUBLIC_IP" == *"Unauthorized"* || -z "$PUBLIC_IP" ]]; then
    PUBLIC_IP="Metadata Unavailable"
fi

if [[ "$HOSTNAME" == *"Unauthorized"* || -z "$HOSTNAME" ]]; then
    HOSTNAME="Metadata Unavailable"
fi

# Overwrite the HTML file with updated content
sudo tee /var/www/html/index.html > /dev/null <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>My Web App</title>
    <style>
        button {
            background-color: #007bff;
            color: white;
            padding: 10px 20px;
            font-size: 16px;
            border: none;
            cursor: pointer;
            border-radius: 5px;
        }
        button:hover {
            background-color: #0056b3;
        }
    </style>
</head>
<body>
    <h1>My Web App is Running!</h1>
    <h2>Instance IP: $PUBLIC_IP</h2>
    <h2>Hostname: $HOSTNAME</h2>
    <hr>
    <h3>Visit My S3-Hosted Website:</h3>
    <button onclick="window.location.href='https://mywafprojectbucket.s3.us-east-1.amazonaws.com/index.html'">
        Go to S3 Website
    </button>
</body>
</html>
EOF

# Restart Apache to apply changes
sudo systemctl restart httpd
  ```

2. Make the script executable
```  sudo chmod +x /usr/local/bin/update_metadata.sh ```

3. Ensure it runs on every reboot:
  -  Open crontab:
 ```    sudo crontab -e   ```

  - Add this line at the bottom:
 ```  @reboot /usr/local/bin/update_metadata.sh   ```

  - Save and exit.

** Results:

https://github.com/user-attachments/assets/19f0d76f-2f04-4a25-bba7-8ddccd4e45ba

### Step 9: Steps to Create an Amazon Machine Image (AMI):

1. Select the EC2 Instance
    - Identify the EC2 instance you want to create an AMI from.
    - Click the checkbox next to the instance.

2. Create an AMI
    - Click on Actions → Image and templates → Create image.
    - Fill in the required details:
    - Image name: (e.g., MyWebServer-AMI)
    - Image description (optional): Add details for reference.
    - No reboot (optional): Check this box if you don’t want the instance to restart.
    - Click Create image.

3. Monitor AMI Creation
    - In the left panel, click AMIs under Images.
    - Locate your AMI and check the Status:
    - Pending: AMI is being created.
    - Available: AMI is ready for use.

4. Launching an EC2 Instance from the AMI.
    - Go to EC2 Dashboard → Click AMIs.
    - Select your AMI and click Launch instance from image.
    - Choose an instance type, configure settings, and click Launch.

![Screenshot 2025-02-26 183233](https://github.com/user-attachments/assets/bb65eae7-cd14-4599-a3be-39e1893b3917)

<br><br>

![Screenshot 2025-02-28 174442](https://github.com/user-attachments/assets/665eefb8-db3b-4487-bc27-9b0fd6a7b484)

<br><br>

![Screenshot 2025-03-01 143045](https://github.com/user-attachments/assets/2e19f815-c66e-4880-89d6-164f0cfaa5f4)

<br><br>

https://github.com/user-attachments/assets/c9f2e686-9ace-4949-966e-f51f0f0db9e2

<br><br>

![Screenshot 2025-03-01 144252](https://github.com/user-attachments/assets/25d08a7a-5268-4dd9-af06-7b9a14ddb23f)

<br><br>


### Step 9: Setting up AUto-Scaling-Groups(ASG):

Step 1: Create a Launch Template.

-  Option 1: From CLI
 ```
aws ec2 create-launch-template \
    --launch-template-name MyWebAppTemplate \
    --version-description "Initial version" \
    --launch-template-data '{
        "ImageId": "<your-ami-id>",
        "InstanceType": "t2.micro",
        "KeyName": "<your-key-pair>",
        "SecurityGroupIds": ["<your-security-group-id>"],
        "IamInstanceProfile": {
            "Name": "<your-iam-instance-profile>"
        },
        "UserData": "'"$(base64 -w 0 user-data.sh)"'"
    }'

 ```
Replace:
```
  - <your-ami-id> → Your newly created AMI ID.
  - <your-key-pair> → Your EC2 key pair.
  - <your-security-group-id> → The security group ID that allows HTTP (port 80).
  - <your-iam-instance-profile> → The IAM role attached to your EC2 instances (must allow S3 access if needed).
```
-  Option 2: From AWS GUI



https://github.com/user-attachments/assets/2761be96-08aa-446e-a07d-b68dea631d8f

<br><br>

![Screenshot 2025-03-01 234735](https://github.com/user-attachments/assets/97e3fb2a-9d7a-4db7-beea-e628346ecf62)

<br><br>

![Screenshot 2025-03-02 000502](https://github.com/user-attachments/assets/0ccb1545-7225-48a2-9a25-ad7a9c9cfee9)

* An Issue faced: I believe that since we have enabled Scale-in permissions to the ASG it's automatically terminating instances after a while, and then again creating a new instance(Basically creating a loop) probably due to low CPU utilization. So let's make changes to it in the ASG settings and see.

   - Indeed, we were on the right path, and performing the steps below resolved the automatic termination of the instances, stopping the loop.

https://github.com/user-attachments/assets/2a5fb456-f00c-4494-97d6-18bc642fd21b

