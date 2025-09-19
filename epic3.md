# Epic 3: Clean up resources

**Important Note:** This part of the walkthrough covers how to clean up all resources created during this demonstration to avoid incurring unnecessary AWS charges.

## Story 1: Delete the CloudFormation stacks


1. In the CloudFormation console.
2. Choose **Stacks**, in the navigation pane, .

3. Delete the ISV solution stack:
   - Select the `ISV-Solution` stack (created in Epic 2).
   - Choose **Delete** and confirm the deletion by choosing **Delete** again.
   - Wait for the stack deletion to complete. This may take a few minutes.

4. Delete the customer database stack:
   - Select the `Customer-MySQL-DB` stack (created in Epic 1).
   - Choose **Delete** and confirm the deletion by choosing **Delete** again.
   - Wait for the stack deletion to complete. This may take 5-10 minutes as the RDS instance is deleted.

5. Verify that all resources have been deleted:
   - Refresh the CloudFormation console to confirm both stacks are no longer listed.
   - You can also check the EC2 console to verify the instance has been terminated.
   - Check the RDS console to verify the database instance has been deleted.
   - Check the Secrets Manager console to verify the secret has been deleted.

**Congratulations!** You have successfully completed the demonstration of the secure ISV integration pattern and cleaned up all associated resources.
