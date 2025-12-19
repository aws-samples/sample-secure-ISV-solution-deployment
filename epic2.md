# Epic 2: Deploy the ISV solution

**Important Note:** This part of the walkthrough represents the ISV solution deployment. In a real-world scenario, an ISV would provide a CloudFormation template that customers deploy in their own AWS accounts, connecting to their existing resources without the ISV needing access to sensitive information.

**Resources Created:**
- EC2 instance running the ISV application
- AWS Secrets Manager secret to store database credentials
- IAM role and instance profile for EC2 to access Secrets Manager and allow Systems Manager Access

## Story 1: Review and understand the ISV CloudFormation template

This procedure helps you understand the key components of the ISV CloudFormation template before deployment. Reviewing the template will give you insight into how the solution securely handles database credentials and connects to your existing resources.

1. Open the `epic2-ec2.yaml` file in a text editor.
2. Review the following key sections:
   - **Parameters**: Note how the template accepts database connection parameters from the customer
   ```yaml
   Parameters:
    MySQLHost:
      Type: String
      Description: Hostname or IP address of the MySQL server (DBEndpoint from Epic 1)
      NoEcho: true

    MySQLUsername:
      Type: String
      Description: Username for MySQL connection (DBUsername from Epic 1)
      NoEcho: true

    MySQLPassword:
      Type: String
      Description: Password for MySQL connection
      NoEcho: true
   ```

   - **Secrets Manager**: Observe how the template creates a secret to store the database credentials
   ```yaml
   MySQLSecret:
     Type: AWS::SecretsManager::Secret
     Properties:
       Name: !Sub "${AWS::StackName}-mysql-credentials"
       Description: "MySQL credentials for EC2 application"
       SecretString: !Sub '{"host":"${MySQLHost}","username":"${MySQLUsername}","password":"${MySQLPassword}","port":"3306"}'
   ```

   - **IAM Role**: See how the EC2 instance is granted permission to access only the specific secret
   ```yaml
   EC2InstanceRole:
     Type: AWS::IAM::Role
     Properties:
       AssumeRolePolicyDocument:
         Version: '2012-10-17'
         Statement:
           - Effect: Allow
             Principal:
               Service: ec2.amazonaws.com
             Action: sts:AssumeRole
       ManagedPolicyArns:
         - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore  # For SSM access
       Policies:
         - PolicyName: SecretsManagerAccess
           PolicyDocument:
             Version: '2012-10-17'
             Statement:
               - Effect: Allow
                 Action:
                   - secretsmanager:GetSecretValue
                 Resource: !Ref MySQLSecret
   ```

   - **User Data**: Examine how the EC2 instance user data is passed the ARN of the secret which it then uses to retrieve the secret to connect to the database
   ```yaml
   UserData: 
     Fn::Base64: !Sub |
       #!/bin/bash
       
       # Add environment variable to /etc/environment
       echo "Adding environment variable to /etc/environment..."
       echo "MYSQL_SECRET_ARN=${MySQLSecret}" >> /etc/environment
       
       # Run the script to test database connection
       sudo -u ec2-user MYSQL_SECRET_ARN="${MySQLSecret}" /home/ec2-user/test-db-connection.py
   ```

   - **Python Script**: The EC2 instance includes a Python script that retrieves credentials from Secrets Manager and tests the database connection. Key parts of this script include:

   Secret retrieval:
   ```python
   def get_secret():
       """Get MySQL credentials from AWS Secrets Manager"""
                       
                       ...
                       
           # Get the secret value
           get_secret_value_response = client.get_secret_value(
               SecretId=secret_arn
           )
           secret = get_secret_value_response['SecretString']
           result["success"] = True
           
           return result, json.loads(secret)

   ```

   Database connection:
   ```python
   def test_db_connection(secret_data):
       """Test connection to MySQL database using credentials from secret"""
       result = {
           "success": False,
           "error": None
       }
       
       if not secret_data:
           result["error"] = "No secret data provided"
           return result
       
       try:
           # Extract connection parameters from secret
           host = secret_data.get('host')
           username = secret_data.get('username')
           password = secret_data.get('password')
           port = int(secret_data.get('port'))
           
           # Connect to the database
           conn = mysql.connector.connect(
               host=host,
               user=username,
               password=password,
               port=port,
               connect_timeout=5
           )
           
           # Close connection
           conn.close()
           result["success"] = True
           
       except mysql.connector.Error as e:
           result["error"] = str(e)
       
       return result
   ```

3. Pay special attention to the following security features:
   - Sensitive parameters are marked with `NoEcho: true` to prevent their values from being displayed or logged
   - The IAM role has least privilege access (only to the specific secret)
   - Credentials are never stored in plain text on the EC2 instance
   - The template is designed by the ISV but the customer enters their data independently

## Story 2: Deploy the ISV solution

This procedure deploys the ISV solution using CloudFormation. Before starting, ensure you have completed Epic 1 and have the database connection information available.

1. In the CloudFormation console.
2. In the navigation pane, choose **Stacks**.
3. Choose **Create stack**, and then choose **With new resources (standard)**.
4. Under **Prerequisite - Prepare template**, select **Choose an existing template**.
5. Under **Specify template**, choose **Upload a template file** and upload the `epic2-ec2.yaml` file.
6. Choose **Next**.
7. For **Stack name**, enter `ISV-Solution`.
8. Under **Parameters**, configure the following:
   - **Network Configuration**:
     - For **PublicSubnetId**, select a public subnet in the VPC you used for the database deployment in Epic 1
   - **Database Configuration**:
     - For **MySQLHost**, enter the **DBEndpoint** value from Epic 1
     - For **MySQLUsername**, enter the **DBUsername** value from Epic 1 (should be `admin`)
     - For **MySQLPassword**, enter the database password you specified in Epic 1
     - For **EC2SecurityGroupId**, enter the **EC2SecurityGroupId** value from Epic 1
   - **EC2 Instance Configuration**:
     - For **EC2InstanceName**, enter `ISV-Application`
9.  Choose **Next**.
10. On the **Configure stack options** page, keep the default settings, acknowledge any capabilities if prompted (the template requires IAM capabilities), and choose **Next**.
11. On the **Review and create** page, review your settings and choose **Submit**.
12. Wait for the stack creation to complete. This may take 5-10 minutes.

## Story 3: Verify the deployment and test connectivity

This procedure helps you verify that the ISV solution has been deployed correctly and can connect to your database. You'll use AWS Systems Manager Session Manager to access the EC2 instance and check the database connection.

1. Once the stack creation is complete, go to the **Outputs** tab of your stack.
2. Note the following information:
   - **InstanceId**: The ID of the EC2 instance
   - **MySQLSecretARN**: The ARN of the secret created in AWS Secrets Manager

3. To access the EC2 instance and verify database connectivity:
   - In the AWS Console, search for and choose **Systems Manager**
   - In the navigation pane, choose **Session Manager**
   - Choose **Start session**
   - Select the EC2 instance you just created (you can find it by the instance ID from the outputs)
   - Choose **Start session** to connect to the instance

4. Once connected to the instance via Session Manager, you need to switch to the ec2-user to access the test files:
   ```
   sudo su ec2-user
   ```

5. Move to the home directory and view the solutions files:
  ```
   cd ~
   ls
   ```

6. You should see the database connection test results file db-connection-story.txt and the Python test script created by the user data test-db-connection.py. View the result of the connection test with:
   ```
   cat db-connection-story.txt
   ```

7. The output will show a narrative of the connection attempt:
   - If the connection is successful, you'll see a success message indicating which secret was used to connect to the database
   - If the connection failed, you'll see an error message with details about what went wrong
   - The narrative includes steps showing the secret retrieval and database connection attempt

8. To verify the secure credential handling:
   - Open the AWS Secrets Manager console
   - Find the secret created by the stack (you can use the MySQLSecretARN from the outputs)
   - Note that you can view the secret values, but the EC2 instance retrieves them securely without storing them in plain text
   - To force a failure, you could modify the secret values in Secrets Manager and then run the test script again on the EC2 instance:
     ```
     python3 ~/test-db-connection.py
     ```

9.  This demonstrates the complete pattern:
   - The ISV provided a solution template
   - You deployed it with your own database credentials
   - The credentials are stored securely in Secrets Manager
   - The application retrieves and uses them securely
   - All of this happens without the ISV ever accessing your sensitive information

## Story 4: Understand the end-to-end process flow

This solution demonstrates a complete secure integration pattern that works as follows:

1. **Customer Deployment**:
   - The customer receives a CloudFormation template from the ISV
   - The customer deploys this template in their own AWS account
   - During deployment, the customer provides their database credentials as parameters
   - These credentials are never exposed to the ISV

2. **Resource Creation**:
   - CloudFormation creates an EC2 instance with appropriate IAM permissions
   - CloudFormation creates a secret in AWS Secrets Manager containing the database credentials
   - The EC2 instance is configured with SSM access for secure management

3. **Automatic Configuration**:
   - When the EC2 instance launches, the user data script runs automatically
   - The script sets the secret ARN as an environment variable
   - The script then runs the Python test script as the ec2-user

4. **Secure Credential Handling**:
   - The Python script retrieves the secret ARN from the environment variable
   - It uses the AWS SDK to securely fetch the credentials from Secrets Manager
   - The credentials are used in memory only, never stored in plain text on disk
   - The script attempts to connect to the database using these credentials
   - Results are written to a text file for verification

This pattern ensures:
- Sensitive information is securely stored and transmitted
- The principle of least privilege is maintained throughout
- The ISV never needs access to customer credentials or environments
- The solution is fully automated, reducing deployment errors
- The customer maintains full control over their credentials and resources
