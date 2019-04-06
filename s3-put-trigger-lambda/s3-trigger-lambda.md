S3にcreateObjectをトリガーにLambdaを起動するCloudformationテンプレート
===

## はじめに
こんにちは、中村です。  

S3にオブジェクトをPutしたタイミングで、Lambdaをキックしたいケースがあるかと思います。手で作成する場合は、Lambda上でトリーが設定することで問題ないですが、ありがちな構成なのでCloudformationで作成しました。

## テンプレート
作成するリソースはこちらです。  

* Lambdaロール
* Lambda
* Lambdaパーミッション
* S3バケット

今回のLambdaは、S3にオブジェクトをPutした後、txtファイルの場合発火しLambdaを起動します。
Lambdaは、S3のファイルの中身を確認します。そのため、ロールには、**S3の読み込み権限・CloudWatchLogsのフル権限**を与えます。  

Lambdaのコードは、S3バケットにzipを元に作成します。このソースをzipしてバケットにPutしておきます。
[javascript]
const Aws = require('aws-sdk');
const S3 = new Aws.S3();

exports.handler = async(event) => {
    console.log(JSON.stringify(event));
    let param = {
        "Bucket": event.Records[0].s3.bucket.name,
        "Key": event.Records[0].s3.object.key
    };

    let data = S3.getObject(param).promise();
    console.log(data.Body.toString());

    return { statusCode: 200 };
}
[/javascript]

パーミッションは、S3イベント通知を登録する前に、適切なアクセス権限が必要です。S3バケット名はRefを利用して参照したいところですが、循環依存になってしまいます。
そのため静的にバケット名を記載します。

[yaml]
---
AWSTemplateFormatVersion: "2010-09-09"
Description: "Test resources."
Resources:
  TriggerLambdaRole:
    Type: "AWS::IAM::Role"
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - "sts:AssumeRole"
      ManagedPolicyArns: 
        - "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        - "arn:aws:iam::aws:policy/CloudWatchLogsFullAccess"
      RoleName: "trigger_lambda_role"
  TriggerLambda:
    Type: "AWS::Lambda::Function"
    Properties:
      Code:
        S3Bucket: "CODE-BUCKET-HERE"
        S3Key: "lambda.zip"
      FunctionName: "trigger_lambda"
      Handler: "index.handler"
      MemorySize: 128
      Role: !GetAtt "TriggerLambdaRole.Arn"
      Runtime: "nodejs8.10"
      Timeout: 5
  TriggerLambdaPermission:
    Type: "AWS::Lambda::Permission"
    Properties:
      Action: "lambda:InvokeFunction"
      FunctionName: !GetAtt 
        - TriggerLambda
        - Arn
      Principal: "s3.amazonaws.com"
      SourceArn: !Join 
                  - ""
                  - - "arn:aws:s3:::"
                    - "TRIGGER-BUCKET-NAME-HERE"
  SrcS3Bucket:
    Type: "AWS::S3::Bucket"
    DependsOn: "TriggerLambdaPermission"
    Properties:
      BucketName: "TRIGGER-BUCKET-NAME-HERE"
      NotificationConfiguration:
        LambdaConfigurations:
          - Event: "s3:ObjectCreated:*"
            Filter:
              S3Key:
                Rules:
                  - Name: suffix
                    Value: txt
            Function: !GetAtt
              - TriggerLambda
              - Arn
[/yaml]

"Hello, World!"と記載したテキストファイルを作成し、アップロードしましょう。
結果をCloudWatchLogsで確認します。

![cwl.png](cwl.png)

## まとめ
いかがでしたでしょうか。地味にハマりそうなので書いておきました。  
手作業でも勿論作成可能ですが、使いそうなものはテンプレ化しておきたいなと思う今日この頃です。

弊社では、「Amazon Connect」の導入を検討している方を対象とした無料相談会を毎週開催中です。

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" title="【東京・大阪】クラウド型コンタクトセンター「Amazon Connect」の導入・運用に関する無料相談会を実施中" src="https://hatenablog-parts.com/embed?url=https://classmethod.jp/news/weekly-connect-consultation/" width="300" height="150" frameborder="0" scrolling="no"></iframe>

また音声を中心とした各種ソリューションの開発支援も行なっております。

<ul>
<li><a href="https://classmethod.jp/services/chatbot/" target="_blank">チャットボット開発支援</a></li>
<li><a href="https://classmethod.jp/services/amazon-connect/" target="_blank">クラウド型コンタクトセンターサービス導入支援</a></li>
</ul>