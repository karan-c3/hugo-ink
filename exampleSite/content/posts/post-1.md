---
title: "Using AWS AppSync client in lambda resolver"
date: 2018-03-18T02:01:58+05:30
description: "In this article, I'll explain how we can execute graphql query & mutations in AWS lambda resolver using [AWSAppSyncClient](https://github.com/awslabs/aws-mobile-appsync-sdk-js)."
tags: [AWS, APPSYNC, javascript, awslambda, graphql, amplify]
---
# Using AWS AppSync client in lambda resolver
In this article, I'll explain how we can execute graphql query & mutations in AWS lambda resolver using [AWSAppSyncClient](https://github.com/awslabs/aws-mobile-appsync-sdk-js).

*Note:- I am not going to write about how to setup amplify & appsync, there are a lot of articles and docs you can start from [Amplify docs](https://docs.amplify.aws/)*

I assume you already have an appsync project you have a custom lambda resolver you want to reuse the autogenerated CRUD queries/mutation from appsync, in my case I have to combine 3 mutations generated by appsync in a single mutation, previously I was directly modifying dynamodb, but it has several drawbacks 

 1. First you have to repeat the code which is already generated by AppSync 
 2. if you have pipeline resolvers or subscriptions on the mutation's it won't work.

So instead I have used the AWSAppSyncClient lambda function for calling existing mutations, just like you would call from frontend clients.

let's start, First add the lambda function using amplify cli
```bash
$ amplify add function
? Select which capability you want to add: Lambda function (serverless function)
? Provide an AWS Lambda function name: <choose your functions name>
? Choose the runtime that you want to use: NodeJS
? Choose the function template that you want to use: Hello World

Available advanced settings:
- Resource access permissions
- Scheduled recurring invocation
- Lambda layers configuration

? Do you want to configure advanced settings? (y/N) N
? Do you want to edit the local lambda function now?  
Yes
```
Now head into the functions root directory i.e. `cd functions/<your functions name>`  
Install following dependencies 
`npm i --save  aws-appsync es6-promise graphql-tag isomorphic-fetch`
or if you use yarn
`yarn add  aws-appsync es6-promise graphql-tag isomorphic-fetch`

open handler file i.e. index.js in your functions directory, import required dependencies
```js
// index.js
require('es6-promise').polyfill()
require('isomorphic-fetch')

import  AWSAppSyncClient, { AUTH_TYPE } from  'aws-appsync'
import  gql  from  'graphql-tag'
```
In the above snippet first, we are importing polyfills for **es6-promise** & **fetch** which is required for **aws-appsync** pkg so that it works in node runtime version 8 or less, we are using graphql-tag to parse graphql queries.

Now we'll initailise our AWSAppSyncClient, we'll be using AMAZON_COGNITO_USER_POOLS for authenticating our client, there other ways to authenticate you can refer https://docs.amplify.aws/cli/auth/overview
```js
// index.js
exports.handler = async (event, context, callback) => {
    const client = new AWSAppSyncClient({
        url: "<your graphql endpoint>",
        region: process.env.AWS_REGION!,
        auth: {
            type: AUTH_TYPE.AMAZON_COGNITO_USER_POOLS,
            jwtToken: event.request.headers.authorization // this is the same token which user passed while calling this API
        },
        disableOffline: true
    });

    const res = await client.mutate({
        mutation: gql`mutation MyMutation($name: String!!) {
                createTodo(name: $name) {
                    id
                    name
                }
            }`,
        variables: {
            name: 'test todo',
        },
    })
    const response: any = {
        statusCode: 200,
        message: JSON.stringify(res.data.createTodo)
    }
    return callback(null, response)
}
```
If you take look at authentication code auth.jwtToken property we are using the same token which user has sent while calling this mutation, I've extracted it from `event.request.headers.authorization` property.

That's it we are done, now just push changes to aws amplify, test it on the console. using following cmd
`amplify push`
it would take few mins after that you can run the following cmd to open the console
`amplify console`

## Full code
```js
require('es6-promise').polyfill()
require('isomorphic-fetch')

import  AWSAppSyncClient, { AUTH_TYPE } from  'aws-appsync'
import  gql  from  'graphql-tag'

exports.handler = async (event, context, callback) => {
    const client = new AWSAppSyncClient({
        url: "<your graphql endpoint>",
        region: process.env.AWS_REGION!,
        auth: {
            type: AUTH_TYPE.AMAZON_COGNITO_USER_POOLS,
            jwtToken: event.request.headers.authorization // this is the same token which user passed while calling this API
        },
        disableOffline: true
    });

    const res = await client.mutate({
        mutation: gql`mutation MyMutation($name: String!!) {
                createTodo(name: $name) {
                    id
                    name
                }
            }`,
        variables: {
            name: 'test todo',
        },
    })
    const response: any = {
        statusCode: 200,
        message: JSON.stringify(res.data.createTodo)
    }
    return callback(null, response)
}

```