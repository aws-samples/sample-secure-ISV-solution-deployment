# Epic 1: Set up the customer environment

**Important Note:** This part of the setup represents the customer's existing environment. In a real-world scenario, customers would already have their database and network infrastructure in place. We're creating these resources for demonstration purposes only.

**Resources Created:**
- MySQL RDS database instance in a private subnet
- EC2 security group for database access
- Necessary IAM roles and permissions

## Story 1: Deploy a sample RDS database stack

1. Sign in to the AWS Management Console.
2. Open the CloudFormation console.
3. In the navigation pane, choose **Stacks**.
4. Choose **Create stack**, and then choose **With new resources (standard)**.
5. Under **Prerequisite - Prepare template**, select **Choose an existing template**.
6. Under **Specify template**, choose **Upload a template file** and upload the `epic1-rds.yaml` file.
7. Choose **Next**.
8. For **Stack name**, enter `Customer-MySQL-DB`.
9. Under **Parameters**, configure the following:
   - For **VpcId**, select an existing VPC that has **both public and private subnets**. This is important for the proper functioning of the solution.
   - For **PrivateSubnetIds**, select at least two private subnets in different Availability Zones from the VPC selected above
   - For **DBName**, enter `demodb`
   - For **DBUsername**, enter `admin`
   - For **DBPassword**, enter a secure password of at least 8 characters and make note of it
10. Choose **Next**.
11. On the **Configure stack options** page, keep the default settings and choose **Next**.
12. On the **Review and create** page, review your settings, acknowledge any capabilities if prompted, and choose **Submit**.
13. Wait for the stack creation to complete. This may take 5-10 minutes.

## Story 2: Note the connection information for your database

1. Once the stack creation is complete, go to the **Outputs** tab of your stack.
2. Note the following important information that you'll need for deploying the ISV solution:
   - **DBEndpoint**: The endpoint address of your MySQL database
   - **DBUsername**: The admin username (should be `admin`)
   - **DBName**: The name of the database (should be `demodb`)
   - **VPCID**: The ID of the VPC where the database is deployed
   - **DBSecurityGroupID**: The ID of the security group created for database access

   **Note**: For security reasons, the database password is not included in the outputs. You should use the password you specified during creation.
3. Create a new text file to store this information securely, as you'll need it when deploying the ISV solution in Epic 2.
