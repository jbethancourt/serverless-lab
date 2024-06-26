# LevelUp! Lab for Serverless

## Lab Overview And High-Level Design

Let's start with the High-Level Design.
![High Level Design](./images/high-level-design.jpg)
An Amazon API Gateway is a collection of resources and methods. For this tutorial, you create one resource (DynamoDBManager) and define one method (POST). The method is backed by a Lambda function (LambdaFunctionOverHttps). That is when you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

* Create, update, and delete an item.
* Read an item.
* Scan an item.
* Other operations (echo, ping), not related to DynamoDB, that you can use for testing.

The request payload you send in the POST request is the DynamoDB operation and provides the necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}
```
The following is a sample request payload for a DynamoDB read item operation:
```json
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
```

## Setup

### Create Lambda IAM Role 
Create the execution role that gives your function permission to access AWS resources.

To create an execution role

1. Open the roles page in the IAM console.
2. Choose Create role.

**Note:**
Create a role with the following properties.
    * Trusted entity – Lambda.
    * Role name – **lambda-apigateway-role**.
    * Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that  the function needs to write data to DynamoDB and upload logs.

3. Click "Create Role" in the AWS IAM Console

![Create role](./images/create-lambda-role.jpg)

4. Select Lambda as "Trusted entity", then Click "Next"

![Create role](./images/select-lambda-trust-entity.jpg)

5. Click "Create role"

![Create role](./images/create-lambda-role-1.jpg)

6. Create a new inline policy for the new role. Click "Create inline policy"

![Create role](./images/create-inline-policy-lambda-role.jpg)

7. Select the "JSON" button. Copy and paste the JSON posted below, into the Policy editor. Click "Next"

![Create role](./images/specify-inline-policy-permissions-lambda-role.jpg)

```json
{
    "Version": "2012-10-17",
    "Statement": [
    {
      "Sid": "Stmt1428341300017",
      "Action": [
        "dynamodb:DeleteItem",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:UpdateItem"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "",
      "Resource": "*",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow"
    }]
}
```
8. Type "DynamoDB_CloudWatch_Logs" inside the Policy Name textbox. Click "Create Policy"

![Create function](./images/create-inline-policy-permissions-lambda-role.jpg)



### Create Lambda Function

**To create the function**
1. Click "Create function" in the AWS Lambda Console

![Create function](./images/create-lambda.jpg)

2. Select "Author from scratch". Use the name **LambdaFunctionOverHttps**, and select **Python 3.7** as Runtime. Under Permissions, select "Use an existing role", and select **lambda-apigateway-role** that we created, from the drop-down

3. Click "Create function"

![Lambda basic information](./images/lambda-basic-info.jpg)

4. Click "Code" under the "Function Overview" section, and replace the boilerplate coding with the following code snippet

**Example Python Code**
```python
from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
```
![Lambda Code](./images/lambda-code-paste.jpg)

### Test Lambda Function

Let's test our newly created function. We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.

1. Click the "Test" option under the "Function Overview" section and select "Configure test event"

![Configure test events](./images/lambda-test-event-create.jpg)

2. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from the input event as output. Click "Save"
```json
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}
```
![Save test event](./images/save-test-event.jpg)

3. Click "Deploy", and it will deploy the lambda function. The deploy button should turn gray.

![Save test event](./images/lambda-deploy-api.jpg)

4. Click "Test", and it will execute the test event. You should see the output in the console.

![Execute test event](./images/execute-test.jpg)

We're all set to create DynamoDB table and an API using our lambda as a backend!

### Create DynamoDB Table

Create the DynamoDB table that the Lambda function uses.

**To create a DynamoDB table**

1. Open the DynamoDB console.
2. Choose Create table.
3. Create a table with the following settings.
   * Table name – lambda-apigateway
   * Primary key – id (string)
4. Choose Create table

![create DynamoDB table](./images/create-dynamo-table.jpg)
![create DynamoDB table](./images/create-dynamo-table-button.jpg)


### Create API

**To create the API**
1. Go to API Gateway console
2. Scroll down and select "Build" for REST API

![Build REST API](./images/build-rest-api.jpg) 

3. Give the API name "DynamoDBOperations", keep everything as is, and click "Create API"

![Create REST API](./images/create-new-api.jpg)

4. Each API is a collection of resources and methods integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next 

Click "Create Resource"

![Create API resource](./images/create-api-resource.jpg)

5. Input "DynamoDBManager" in the Resource Name, Resource Path will get populated. Click "Create Resource"

![Create resource](./images/create-resource-name.jpg)

6. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Create Method". 

![Create resource method](./images/create-method-1.jpg)

7. Select "POST" from the drop-down. Select "Lambda function" from the "Integration type" section. Select "LambdaFunctionOverHttps" function that we created earlier from the "Lambda function" section. Scroll down and click "Create method".

![Create resource method](./images/create-method-2.jpg)
![Create resource method](./images/create-method-3.jpg)

Our API-Lambda integration is done!

### Deploy the API

In this step, you deploy the API you created to a stage called prod.

1. Click "Deploy API"

![Create lambda integration](./images/deploy-api-1.jpg)

2. Now it is going to ask you about a stage. Select "[New Stage]" for "Deployment stage". Give "Prod" as "Stage name". Click "Deploy"

![Deploy API to Prod Stage](./images/deploy-api-2.jpg)

3. We're all set to run our solution! To invoke our API endpoint, we need the endpoint URL. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from the screen

![Copy Invoke Url](./images/copy-invoke-url.jpg)


### Running our solution

1. The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
```
2. To execute our API from the local machine, we are going to use Postman and Curl commands. You can choose either method based on your convenience and familiarity. 
    * To run this from Postman, select "POST", and paste the API invoke url. Then under "Body" select "raw" and paste the above JSON. Click "Send". API should execute and return "HTTPStatusCode" 200.

    ![Execute from Postman](./images/create-from-postman.jpg)

    * To run this from the terminal using Curl, run the below
    ```
    $ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
    ```   
3. To validate that the item is indeed inserted into the DynamoDB table, go to the Dynamo console, select "lambda-apigateway" table, select the "Explorer Items" option and the newly inserted item should be displayed.

![Dynamo Item](./images/dynamo-item.jpg)

4. To get all the inserted items from the table, we can use the "list" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table

```json
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```
![List Dynamo Items](./images/dynamo-item-list.jpg)

5. To get specific inserted items from the table, we can use the "read" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table

```json
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "4321ABCD"
        }
    }
}
```
![List Dynamo Items](./images/dynamo-item-read.jpg)

6. To update specific inserted items from the table, we can use the "update" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table

```json
{
    "operation": "update",
    "tableName": "lambda-apigateway",
    "payload": {    
        "Key" : {"id": "1234ABCD"},
        "UpdateExpression" : "set #number = :n",
        "ExpressionAttributeNames" : {
            "#number": "number"
        },
        "ExpressionAttributeValues" : {
            ":n": 20
        },
        "ReturnValues" : "UPDATED_NEW"
    }
}
```
![List Dynamo Items](./images/dynamo-item-update.jpg)

6. To delete specific inserted items from the table, we can use the "delete" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table

```json
{
    "operation": "delete",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "4321ABCD"
        }
    }
}
```
![List Dynamo Items](./images/dynamo-item-delete.jpg)

We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

## Cleanup

Let's clean up the resources we have created for this lab.


### Cleaning up DynamoDB

To delete the table, from the DynamoDB console, select the table "lambda-apigateway", and click "Delete"

![Delete Dynamo](./images/delete-dynamo.jpg)

To delete the Lambda, from the Lambda console, select lambda "LambdaFunctionOverHttps", click "Actions", then click Delete 

![Delete Lambda](./images/delete-lambda.jpg)

To delete the API we created, in the API gateway console, under APIs, select "DynamoDBOperations" API, click "Delete"

![Delete API](./images/delete-api.jpg)
