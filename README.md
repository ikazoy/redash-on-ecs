# Overview
This repository have files to launch Redash and [Redash slack bot](https://github.com/yamitzky/redashbot) on AWS ready for production.

# Files

## cfn-templates/infrasstructure

Cloudformation templates which referred [this repository](https://github.com/aws-samples/ecs-refarch-cloudformation)

These cfn templates set up VPC, security groups, ALB and ECS cluster.

## cfn-template/service

redash-app.yml defines necessary resources specific to Redash application, such as RDS(Postgres), ElastiCache(Redis), ECS service and ALB listener etc.

## application container

There are `docker-compose.yml` and `ecs-params.yml` each for Redash app and [redash bot](https://github.com/yamitzky/redashbot).

As described below, until you explicitly set a value true to launch a redash bot, no related resource will be created.

# Prerequisite
## Charge
You should be charged according to the size of instances and duration you run the stack
- Redash App ECS: t3.medium
- Redash Bot ECS: t2.micro
- RDS: db.t2.micro
- Cache: cache.t2.small
- etc..

## Domain and SSL
You are supposed to have domain managed by Route53 and ACM for it.
And http request to the domain will be redirected to https by ALB.

- domain for redash in your mind managed on Route53 e.g. redash.example.com
- ACM SSL certification on certification for the Redash domain above

## SSH Key to connect to EC2 instances of Redash
In order to debug deeply in case any issues happens and to operate adhoc commands directly (e.g. migration of database), EC2 instances of Redash app are supposed to be launched in public subnet.

Make sure you have a key file like example.pem to connect to the EC2 instances.

Please note you have to modify the security group's rule attached to the EC2 instances to allow ssh connection, which is disabled by default for security.

`ssh -i /path/to/example.pem ec2-user@xxxx.xxxx.xxx.xxx`

# How to launch stacks of Redash app and Redash bot

The steps assumes you are launching Redash for `staging` environment. You can replace `staging` into `production` or whatever you like.

1. Decide a name of this Cloudformation cluster and update `ecs-params-redash-app.yml` and `ecs-params-redash-bot.yml` with the name
2. Set up a string for Redash database with `RedashDbPassword` Name in AWS Systems Manager -> Parameter Store.
   - The Value you set there will be used as password of RDS Postgres instance.
3. Update `.env.staging`
  - `$ cp .env.example .env.staging` and edit it.
  - Set your primary AWS region in `AWS_REGION`.
  - Create a CloudWatch log group in AWS CloudWatch Log console and fill `AWS_LOG_GROUP` with the name of the log group.
    - e.g. ecs-redash
  - For email settings, if you use AWS SES to send email, [this topic](https://discuss.redash.io/t/email-configuration-with-ses/122/15) is good to read.
    - You can get a SMTP credential by follwing [this](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/smtp-credentials.html) and fill `REDASH_MAIL_USERNAME` and `REDASH_MAIL_PASSWORD`.
  - If you want to set your own environment variables for Redash, see [this doc](https://redash.io/help/open-source/admin-guide/env-vars-settings/) and populate whatever values you want in `.env`.
4. Install ecs-cli by following [this](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html) and create a task definition of Redash app ECS service.
   - Decide a name of cloudformation stack which you will create and keep it in your mind.
   - Update ecs-params-redash-app.yml with the name of cloudformation stack.
   - cp .env.staging .env && ecs-cli compose --project-name redash-app-staging --ecs-params ecs-params-redash-app.yml -f docker-compose-redash-app.yml create
5. Create a Cloudformation from AWS Web console with the name you have decided
  - Upload files under a S3 bucket. And put the URL of S3 bucket in `TemplateBaseURL` parameter of the stack.
  - Check the ARN of the task definition you created before in AWS ECS console and put it in `RedashAppTaskDefinitionArn` parameter of the stack.
    - e.g. `arn:aws:ecs:ap-northeast-1:0000000000:task-definition/redash-app-staging:1`
  - Supply other parameters with a proper format by checking the default value and the description of each
  - Keep `ShouldRedashBotLaunched` `false` still
6. Access your <<RedashDomainName>> to confirm redash is now running.
7. If you don't need a Redash slack bot running, you have done! The below steps is only for those who want to run the bot.
8. When the redash app get ready, set these variables in `.env`
   - REDASH_API_KEY
     - Access https://{YOUR_REDASH_HOST}/users/me and retrive a API key.
   - SLACK_BOT_TOKEN
     - Token for the slack bot. e.g. xoxb-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
9. Create a task definition of Redash slack bot ECS service.
  - ecs-cli compose --project-name redash-bot-staging --ecs-params ecs-params-redash-bot.yml -f docker-compose-redash-bot.yml create
10. Update the Cloudformation stack's template.
  - `ShouldRedashBotLaunched` is now `true`.
  - Fill `RedashBotTaskDefinitionArn` with the ARN of the newly created task definition for Redash slack bot in Step #8.

# Links

https://docs.aws.amazon.com/ja_jp/AmazonECS/developerguide/cmd-ecs-cli-compose-parameters.html
https://docs.aws.amazon.com/ja_jp/AmazonECS/developerguide/cmd-ecs-cli-compose-ecsparams.html
https://discuss.redash.io/t/redash-on-aws-ecs/1124/3
https://github.com/yamitzky/redashbot
https://redash.io/help/open-source/admin-guide/env-vars-settings/
