# Serverless file upload to S3

## 동작 구조
![기본 동작 구조](/design.png)

1. 먼저 lambda 서버에 s3의 올릴 url을 요청 한다.
1. 업로드 url을 받는다.
1. 받은 url에다가 파일을 보내 준다.

위와 같이 업로드를 진행 해서 자바스크립트 코드에 aws의 인증키를 노출하지 않고 S3에 바로 업로드를 진행 할 수 있다.

## Serverless 설치

```bash
$ npm i serverless -g
```

## Serverless 기본 프로젝트 만들기
```bash
$ serverless create --template aws-nodejs -p serverless-file-upload
Serverless: Generating boilerplate...
Serverless: Generating boilerplate in "/home/gyuha/workspace/serverless-file-upload"
 _______                             __
|   _   .-----.----.--.--.-----.----|  .-----.-----.-----.
|   |___|  -__|   _|  |  |  -__|   _|  |  -__|__ --|__ --|
|____   |_____|__|  \___/|_____|__| |__|_____|_____|_____|
|   |   |             The Serverless Application Framework
|       |                           serverless.com, v1.30.3
 -------'
$ cd serverless-file-upload
```

## 필요한 패키지 추가해 주기

```bash
$ npm install --save aws-sdk
```
여기서는 s3용 주소를 받기 위해서 aws-sdk를 설치 한다.

### Serverless Offline 설정 하기

개발 할 때 마다 lambda에 올려서 테스트 하기 번거롭다.. aws에서 제공하는 SAM도 있지만, docker를 사용해서 동작하는 구조라서 지나치게 느려서 serverless의 offline plugin을 사용했다.
먼저 패키지를 추가해 준다.

```bash
$ npm install serverless-offline --save-dev
```

`serverless.yml` 파일에 아래 내용을 추가 해 줍니다.

```yml
plugins:
  - serverless-offline
```

### handler.js 파일 작성 하기
```js
'use strict';
const AWS = require('aws-sdk');

module.exports.s3upload = async (event, context) => {
  const s3 = new AWS.S3();
  const params = JSON.parse(event.body);

  const s3Params = {
    Bucket: 'serverless-upload-qwer1',
    Key: params.name,
    ContentType: params.type,
    ACL: 'public-read'
  }

  const uploadURL = s3.getSignedUrl('putObject', s3Params)

  const response = {
    statusCode: 200,
    headers: {
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      uploadURL
    }),
  };
  return response
};
```

## 동작 테스트 하기
```bash
$ serverless offline start
```
로컬 서버를 띄우고.. 아래과 같이 실행해서 정상적으로 문장이 나오면 됨.
```bash
$ curl -d '{"file": "test.jpg", "type": "image/jpeg"}' -H "Content-Type: application/json" -X POST http://localhost:3000/s3upload
```

### 웹브라우저 테스트
파이썬이 깔려 있다면, 아래와 같이 간단하게 서버를 만들 수 있다.
```bash
$ python3 -m http.server 9000
```
간단하게 아래와 같이 접속해서 파일 업로드를 테스트 해 보면 된다.
> http://127.0.0.1:9000/s3upload


## S3및 lambda의 권한 주기
아래는 테스트를 위해서 권한을 지나치게 주었다. 실 사용시에는 조정해서 사용해야 한다.

아래 내용은 `Serverless.yml` 파일 이다.
```yml
service: s3imageupload

plugins:
  - serverless-offline

provider:
  name: aws
  runtime: nodejs8.10
  region: ap-northeast-2
  iamRoleStatements:
    - Effect: "Allow"
      Action:
        - "s3:*"
      Resource: "arn:aws:s3:::serverless-upload-qwer1/*"

functions:
  s3upload:
    handler: handler.s3upload
    events:
      - http:
          method: post
          path: s3upload
          cors:
            origins:
              - '*'
            headers:
              - '*'
            allowCredentials: false

resources:
  Resources:
    ImageUpload:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: serverless-upload-qwer1
        AccessControl: PublicRead
        CorsConfiguration:
          CorsRules:
          - AllowedMethods:
            - GET
            - PUT
            - POST
            - HEAD
            AllowedOrigins:
            - "*"
            AllowedHeaders:
            - "*"
```
S3의 `BucketName`은 기존에 사용하고 있는 이름이면 안 되는 경우가 있다. 중복되지 않도록 이름 짓기를 해야 한다.

## 배포 하기
```bash
$ sls deploy
```

## 로그 확인 하기
```bash
$ sls logs -f <function_name> --tail
```

## 참고
* [Serverless File Uploads](https://www.netlify.com/blog/2016/11/17/serverless-file-uploads/)
* [vue-s3-dropzone](https://github.com/kfei/vue-s3-dropzone)
* [Your CORS and API Gateway survival guide](https://serverless.com/blog/cors-api-gateway-survival-guide/)