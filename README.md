# CrossAccountEventHandlingDemo
Demonstrates the use of cross account event handling using the example of events for branch changes of an AWS CodeCommit being forwarded to spoke accounts.

## Prerequisites
* Two or more AWS accounts in the same AWS Organizations OU
  * Account 1 is considered the Hub account and contains an AWS CodeCommit repository
  * All other accounts are considerd Spoke accounts and son't need to contain any specific resources
* AWS Organizations OU ID is known as <OU_ID>
* AWS CodeCommit repository name is known as <RepositoryName>
* AWS CodeCommit repository branch name is known as <BranchName>
* AWS CodeCommit region is known as <Region>
* Hub account number is known as <HubAccountNo>

## Deployment
### Manual deployment in each account
This deployment method will require the user to go into each account and deploy the template manually
1. Sign in to AWS and switch to the Hub account and Region <Region>.
1. In the CloudFormation console -> Stacks, create a new Stack using the template from the source and name it: e.g. *CrossAccountEventHandlingDemo*
    1. Set Parameters:
        | Parameter | Description | Value | Default Value |
        | --------- | ----------- | ----- | ------------- |
        | BranchName | Name of the CodeCommit branch to monitor for code changes | <BranchName> | N/A |
        | CrossAccountNotificationEventBusName | Custom name for the Event bus receiving events from the Hub Account | | CrossAccountNotification |
        | HubAccount | Account number of the Hub Account with CodeCommit | <HubAccountNo> | N/A |
        | HubRegion | Region of the Hub Account with CodeCommit | <Region> | N/A |
        | NotificationConfigurationTableName | Custom name for the Table containing the configuration of spoke accounts event buses to reveice events from the Hub Account | | NotificationConfiguration |
        | NotificationSubscriptionEventBusName | Custom name for the Event bus receiving subscription events from the Spoke Account | | NotificationSubscription |
        | PrincipleOrgId | Principle Organizational ID that contains all accounts | <OU_ID> | N/A |
        | RepositoryName | Name of the CodeCommit repository | <RepositoryName> | N/A |
        | ResourcePrefix | Prefix for all resources to better identify related resources | | <BLANK> |
    1. Wait for the stack to deploy before moving on to the next step.
1. Switch to the first Spoke account and Region <Region>.
1. In the CloudFormation console -> Stacks, create a new Stack using the template from the source and name it: e.g. *CrossAccountEventHandlingDemo*
    1. Set Parameters identically as for the Hub account:
        | Parameter | Description | Value | Default Value |
        | --------- | ----------- | ----- | ------------- |
        | BranchName | Name of the CodeCommit branch to monitor for code changes | <BranchName> | N/A |
        | CrossAccountNotificationEventBusName | Custom name for the Event bus receiving events from the Hub Account | | CrossAccountNotification |
        | HubAccount | Account number of the Hub Account with CodeCommit | <HubAccountNo> | N/A |
        | HubRegion | Region of the Hub Account with CodeCommit | <Region> | N/A |
        | NotificationConfigurationTableName | Custom name for the Table containing the configuration of spoke accounts event buses to reveice events from the Hub Account | | NotificationConfiguration |
        | NotificationSubscriptionEventBusName | Custom name for the Event bus receiving subscription events from the Spoke Account | | NotificationSubscription |
        | PrincipleOrgId | Principle Organizational ID that contains all accounts | <OU_ID> | N/A |
        | RepositoryName | Name of the CodeCommit repository | <RepositoryName> | N/A |
        | ResourcePrefix | Prefix for all resources to better identify related resources | | <BLANK> |
    1. Wait for the stack to deploy before moving on to the next step.
1. Repeat steps 3 and 4 for any spoke account

## Usage
Deployment of the stack has already configured all relevant resources and implicitly also performed some cross account event handling for the subscription of Spoke account event buses in the Hub account:
1. Switch to the Hub account and Region <Region>.
1. In the DynamoDB console -> PartiQL editor, run the following query:
   ```sql
   SELECT * FROM "<NotificationConfigurationTableName>"
   ```
   where *<NotificationConfigurationTableName>* represents the actual value specified for parameter *NotificationConfigurationTableName* in the deployed stack in the Hub account.
   You should be able to see Items returned similar to:
   | RepositoryArn | Destinations |
   | ------------- | ------------ |
   | arn:aws:codecommit:us-east-1:111122223333:test | [ { "S" : "arn:aws:events:us-east-1:222233334444:event-bus/test" } ] |
This configuration is used to delivery any events generated by the specific *RepositoryArn* to the specified destinations.

Now, push code to the branch <BranchName> of the CodeCommit repository with name <RepositoryName>.
The repository generates an event that triggers the rule *CodeCommitEventRule* on the default Event Bus invoking the *HubEventDeliveryFunction*, which uses the configuration in the DynamoDB table *<NotificationConfigurationTableName>* to forward the event to the specified Destinations. 
**Important**: the new events have a slightly different *Source* (*crossaccount.codecommit* versus *aws.codecommit*), because custom events may not use the *aws* namespace.
To verify the events were received successfully follow these steps:
1. Switch to the first Spoke account and Region <Region>.
1. In the CloudWatch console -> Log groups, open */aws/lambda/<ResourcePrefix>Demo* where *<ResourcePrefix>* represents the spefified value for parameter *ResourcePrefix* in the deployed stack in the Spoke account.
    1. Go to the latest Log Stream. You may need to wait for a few moments for CloudWatch to log all information.
    1. There should be a log entry that looks similar to:
       ```bash
       [INFO] <TimeStamp> <RequestId> event: {...
       ```
       The JSON string following *event:* is the event that originated in the Hub account. Notice the *source* element reading *crossaccount.codecommit* versus *aws.codecommit*.
       You can use an online JSON Formatter such as [JSON Formatter & Validator](https://jsonformatter.curiousconcept.com/) to pretty print the JSON string for better readability.