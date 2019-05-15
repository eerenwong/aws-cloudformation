# AWS Cloudformation
- Cognito user pool cloudformation serverless example
```
resources:
  CognitoSNSRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - cognito-idp.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Policies:
        - PolicyName: 'CognitoSNSPolicy-${self:custom.appName}'
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: 'sns:publish'
                Resource: '*'
  CognitoAuthorizedRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Federated: cognito-identity.amazonaws.com
            Action:
              - 'sts:AssumeRoleWithWebIdentity'
            Condition:
              StringEquals:
                'cognito-identity.amazonaws.com:aud':
                  Ref: CognitoIdentityPool
              'ForAnyValue:StringLike':
                'cognito-identity.amazonaws.com:amr': authenticated
      Policies:
        - PolicyName: 'CognitoAuthorizedPolicy-${self:custom.appName}'
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'mobileanalytics:PutEvents'
                  - 'cognito-sync:*'
                  - 'cognito-identity:*'
                Resource: '*'
              - Effect: Allow
                Action:
                  - 'lambda:InvokeFunction'
                Resource: '*'
              - Effect: Allow
                Action: 's3:*'
                Resource:
                  - 'Fn::Join':
                      - ''
                      - - >-
                          arn:aws:s3:::${file(.env/env.${self:provider.stage}.yml):${self:provider.stage}.BucketName}/
                        - '${cognito-identity.amazonaws.com:sub'
                        - '}/*'
                  - 'Fn::Join':
                      - ''
                      - - >-
                          arn:aws:s3:::${file(.env/env.${self:provider.stage}.yml):${self:provider.stage}.BucketName2}/
                        - '${cognito-identity.amazonaws.com:sub'
                        - '}/*'
              - Effect: Allow
                Action:
                  - 'appsync:GraphQL'
                Resource:
                  'Fn::GetAtt':
                    - GraphQlApi
                    - Arn
  CognitoUnAuthorizedRole:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Federated: cognito-identity.amazonaws.com
            Action:
              - 'sts:AssumeRoleWithWebIdentity'
            Condition:
              StringEquals:
                'cognito-identity.amazonaws.com:aud':
                  Ref: CognitoIdentityPool
              'ForAnyValue:StringLike':
                'cognito-identity.amazonaws.com:amr': unauthenticated
      Policies:
        - PolicyName: 'CognitoUnauthorizedPolicy-${self:custom.appName}'
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'mobileanalytics:PutEvents'
                  - 'cognito-sync:*'
                Resource: '*'
  IdentityPoolRoleMapping:
    Type: 'AWS::Cognito::IdentityPoolRoleAttachment'
    Properties:
      IdentityPoolId:
        Ref: CognitoIdentityPool
      Roles:
        authenticated:
          'Fn::GetAtt':
            - CognitoAuthorizedRole
            - Arn
        unauthenticated:
          'Fn::GetAtt':
            - CognitoUnAuthorizedRole
            - Arn
  UserPoolLambdaInvokePermission:
    Type: 'AWS::Lambda::Permission'
    Properties:
      Action: 'lambda:invokeFunction'
      Principal: cognito-idp.amazonaws.com
      FunctionName: >-
        arn:aws:lambda:ap-southeast-1:${file(.env/env.${self:provider.stage}.yml):${self:provider.stage}.ACCOUNT_ID}:function:functionName-${self:provider.stage}
      SourceArn:
        'Fn::GetAtt':
          - CognitoUserPool
          - Arn
  CognitoUserPool:
    Type: 'AWS::Cognito::UserPool'
    DeletionPolicy: Retain
    Properties:
      SmsConfiguration:
        SnsCallerArn:
          'Fn::GetAtt':
            - CognitoSNSRole
            - Arn
      UserPoolName: '${self:custom.appName}'
      AliasAttributes:
        - email
        - phone_number
      AutoVerifiedAttributes:
        - email
        - phone_number
      Schema:
        - Name: name
          AttributeDataType: String
          Mutable: true
          Required: true
        - Name: phone_number
          AttributeDataType: String
          Mutable: true
          Required: true
        - Name: email
          AttributeDataType: String
          Mutable: true
          Required: false
      LambdaConfig:
        CustomMessage: >-
          arn:aws:lambda:ap-southeast-1:${file(.env/env.${self:provider.stage}.yml):${self:provider.stage}.ACCOUNT_ID}:function:functionName-${self:provider.stage}
      MfaConfiguration: OPTIONAL
      AdminCreateUserConfig:
        AllowAdminCreateUserOnly: false
        UnusedAccountValidityDays: 7
      EmailConfiguration:
        EmailSendingAccount: DEVELOPER
        SourceArn: >-
          ${file(.env/env.${self:provider.stage}.yml):${self:provider.stage}.FROM_EMAIL_ARN}
      Policies:
        PasswordPolicy:
          MinimumLength: 8
          RequireLowercase: true
          RequireNumbers: true
          RequireSymbols: false
          RequireUppercase: true
  CognitoIdentityPool:
    Type: 'AWS::Cognito::IdentityPool'
    DeletionPolicy: Retain
    Properties:
      IdentityPoolName: 'Identity Example ${self:provider.stage}'
      AllowUnauthenticatedIdentities: false
      CognitoIdentityProviders:
        - ClientId:
            Ref: CognitoUserPoolClient
          ProviderName:
            'Fn::GetAtt':
              - CognitoUserPool
              - ProviderName
  CognitoUserPoolClient:
    Type: 'AWS::Cognito::UserPoolClient'
    DeletionPolicy: Retain
    Properties:
      ClientName: 'client-${self:custom.appName}'
      UserPoolId:
        Ref: CognitoUserPool
      ExplicitAuthFlows:
        - ADMIN_NO_SRP_AUTH
      GenerateSecret: false
```
