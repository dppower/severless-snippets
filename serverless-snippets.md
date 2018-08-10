### Api setup with AWS SAM template

An example of a serverless template defined with AWS SAM syntx, to compare with  
serverless.yml. The api defintions are included in a separate yaml file, they are based  
on swagger2.0, the include transform, substitutes in variables, such as the names for lambda  
functions and AWS IAM roles.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Handler: index.handler
    Runtime: nodejs8.10
    Environment:
      Variables:
        LEVEL_NOTES_TABLE: !Ref LevelNotesTableName
    
Parameters:
  ArtifactBucket:
    Type: String
    Description: Name of the bucket to store pipeline ArtifactStore and source zip package
  ApiStageName:
    Type: String
    Description: Name of the api stage
  NotesLambdaRoleArn:
    Type: String
    Description: The arn of the IAM role for notes lambda function
  ApiDatabaseRoleArn:
    Type: String
    Description: The arn of the IAM role for notes lambda function
  LevelDataTableName:
    Type: String
    Description: The name of the level data table
  LevelNotesTableName:
    Type: String
    Description: The name of the level notes table
  LevelTimesTableName:
    Type: String
    Description: The name of the level times table
  LevelVotesTableName:
    Type: String
    Description: The name of the level votes table

Resources:
  ApiGateway:
    Type: AWS::Serverless::Api
    Properties:
      StageName: !Ref ApiStageName
      EndpointConfiguration: REGIONAL
      DefinitionBody:
        'Fn::Transform':
          Name: 'AWS::Include'
          Parameters:
            Location: !Sub "s3://${ArtifactBucket}/api-definitions.yaml"

  GetNotesFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ../src/get-notes/      
      Role: !Ref NotesLambdaRoleArn
      Events:
        GetNotes:
          Type: Api
          Properties:
            RestApiId: !Ref ApiGateway
            Path: /get-notes
            Method: GET

  PostNotesFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ../src/post-notes/      
      Role: !Ref NotesLambdaRoleArn
      Events:
        GetNotes:
          Type: Api
          Properties:
            RestApiId: !Ref ApiGateway
            Path: /post-notes
            Method: POST
       
Outputs:
  ApiEndpoint:
    Description: URL of your API endpoint
    Value: !Sub "https://${ApiGateway}.execute-api.${AWS::Region}.amazonaws.com/${ApiStageName}"
```

### Apigateway DynamoDB direct calls

Apigateway can be used to directly call the DynamoDB api, no need to include a lambda.  
In this sample, an item is put in the ddb table, and the response is transformed using  
VTL. This can result in faster api calls, but error handling and retry logic is harder to  
implement or non-existent.  

```yaml
# api-definitions.yaml
/save-level:
    post:
      consumes:
      - "application/json"
      produces:
      - "application/json"
      responses:
        "200":
          description: "200 response"
          schema:
            $ref: "#/definitions/Success"
        "400":
          description: "400 response"
          schema:
            $ref: "#/definitions/Error"
        "500":
          description: "500 response"
          schema:
            $ref: "#/definitions/Error"
      security:
      - sigv4: []
      x-amazon-apigateway-integration:
        credentials: 
          Ref: ApiDatabaseRoleArn
        uri: 
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:dynamodb:action/PutItem"
        responses:
          default:
            statusCode: "200"
            responseTemplates:
              application/json: |
                {
                  "status": "success",
                  "data": null
                }
          "400":
            statusCode: "400"
            responseTemplates:
              application/json: |
                {
                  "status": "error",
                  "message": $input.json('$.message'),
                  "type": "$input.path('$.__type').split('#')[1]"
                }
          "500":
            statusCode: "500"
            responseTemplates:
              application/json: |
                {
                  "status": "error",
                  "message": $input.json('$.message'),
                  "type": "$input.path('$.__type').split('#')[1]"
                }
        passthroughBehavior: "never"
        httpMethod: "POST"
        requestTemplates:
          application/json: 
            Fn::Sub: |
              {
                "TableName": "${LevelDataTableName}",
                "Item": {
                  "user_id": {"S": "$context.identity.cognitoIdentityId"},
                  "level_name": {"S": "$input.params('level_name')"},
                  "data": {"S": "$util.escapeJavaScript($input.json('$.data'))"}
                }
              }
        type: "aws"
```

### Defining Api Gateway methods in cloudformation

This is the full cloudformation definition for an api-gateway method. It is ridiculously  
verbose when usually SAM can set up a method with 3 or 4 lines. It does give the most control  
and necessary for some api methods e.g. when you want to return both json and plain text from  
api for whatever reason :)

```yaml
SaveChartMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      ApiKeyRequired: true
      AuthorizationType: NONE
      HttpMethod: POST
      ResourceId: !Ref SaveChartResource
      RestApiId: !Ref RestApi
      RequestModels:
        "application/json": !Ref ChartOptionsModel
      MethodResponses:
        - StatusCode: "200"
          ResponseModels:
            "application/json": !Ref UrlModel
            "text/plain": !Ref PlainTextModel
          ResponseParameters:
            "method.response.header.Content-Type": true
        - StatusCode: "400"
          ResponseModels:
            "application/json": !Ref ErrorMessageModel
            "text/plain": !Ref PlainTextModel
          ResponseParameters:
            "method.response.header.Content-Type": true
        - StatusCode: "500"
          ResponseModels:
            "application/json": !Ref ErrorMessageModel
            "text/plain": !Ref PlainTextModel
          ResponseParameters:
            "method.response.header.Content-Type": true
      RequestValidatorId: !Ref BodyValidator
      Integration:
        Type: AWS
        Uri: 
          Fn::Sub: arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${PhantomLambda.Arn}/invocations
        RequestTemplates:
          "application/json": |
                #set($allParams = $input.params())
                {
                    "body" : $input.json('$'),
                    "params" : {
                        #foreach($type in $allParams.keySet())
                        #set($params = $allParams.get($type))
                        "$type" : {
                            #foreach($paramName in $params.keySet())
                            "$paramName" : "$util.escapeJavaScript($params.get($paramName))"#if($foreach.hasNext),#end
                            #end
                        }
                        #if($foreach.hasNext),#end
                        #end
                    }
                }
        IntegrationResponses:
          - StatusCode: "200"
            ResponseParameters:
              "method.response.header.Content-Type": "integration.response.header.Accept"
            ResponseTemplates:
              "application/json": |
                    {
                        "url": $input.json('$')
                    }
              "text/plain": "$input.path('$')"
          - SelectionPattern: "^Invalid.*$"
            StatusCode: "400"
            ResponseParameters:
              "method.response.header.Content-Type": "integration.response.header.Accept"
            ResponseTemplates:
              "application/json": |
                    {
                        "message": $input.json('$.errorMessage')
                    }
              "text/plain": "$input.path('$.errorMessage')"
          - SelectionPattern: "^Error.*$" 
            StatusCode: "500"
            ResponseParameters:
              "method.response.header.Content-Type": "integration.response.header.Accept"
            ResponseTemplates:
              "application/json": |
                    {
                        "message": $input.json('$.errorMessage')
                    }
              "text/plain": "$input.path('$.errorMessage')"
        PassthroughBehavior: NEVER
        IntegrationHttpMethod: POST
        ContentHandling: CONVERT_TO_TEXT
```

### Typing a Lambda function written in javascript

Typescript types can be used to type check javascript in VSCode, this example works  
in the newest version with typescript 2.9.

```typescript
// types.d.ts
interface LambdaProxyResponse {
    headers?: { [header: string]: string };
    statusCode: number;
    body?: string;
}

interface ProxyRequestEvent {
    httpMethod: "GET" | "POST" | "DELETE",
    resource: string;
    path: string;
    headers: {
        [key: string]: string
    },
    queryStringParameters?: {
        [key: string]: string
    },
    pathParameters?: {
        [key: string]: string
    },
    stageVariables?: {
        [key: string]: string
    },
    requestContext: {
        resourceId: string;
        resourcePath: string;
        httpMethod: "GET" | "POST" | "DELETE";
        extendedRequestId: string;
        requestTime: string;
        path: string;
        accountId: string;
        protocol: string;
        stage: string;
        requestTimeEpoch: number;
        requestId: string;
        apiId: string;
        identity: {
            user: string;
            cognitoIdentityId: string;
            cognitoIdentityPoolId: string;
            accountId: string;
            accessKey: string;
            sourceIp: string;
            cognitoAuthenticationProvider: string;
            cognitoAuthenticationType: "authenticated"; 
        }
    },    
    body: string;
    isBase64Encoded: boolean;
}
```

```javascript
/**
 * @param {string} level_id
 * @param {ProxyRequestEvent} event
 * @returns {Promise<LambdaProxyResponse>}
 **/
async function PostNotes(level_id, event) {
    //...

    try {
        // ...
        return {
            statusCode: 200,
            body: JSON.stringify({ status: "success", data: null })
        };
    }
    catch (e) {
        return {
            statusCode: 503,
            body: JSON.stringify({ status: "error", message: "Could not save the notes, try again later." })
        }
    } 
}

/**@param {ProxyRequestEvent} event*/
exports.handler = async(event, context) => {
    
    let query_params = event.queryStringParameters;
    let level_id = query_params && query_params.level_id;

    if (!level_id) {
        return {
            statusCode: 400,
            body: JSON.stringify({ status: "fail", data: "Include the level id as a query parameter (level_id)" })
        };
    }

    return PostNotes(level_id, event);
};
```