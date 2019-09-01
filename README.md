[![CircleCI](https://circleci.com/gh/Exporo-AG/exporo_sls_laravel.svg?style=svg)](https://circleci.com/gh/Exporo-AG/exporo_sls_laravel)

# Exporo Serverless Laravel Boilerplate   

##### Table of Contents  
[Summary](#summary)  
[Requirements](#requirements)  
[Installation](#installation)  
[Deployment](#deployment)  
[Assets](#assets)  
[Local development](#local)  
[Demo application](#demo)  
[Migrate your application](#migration)  
[CI/CD](#CICD)  
[Credits](#credits)  


## Summary
<a name="summary"/>

We are currently developing a boilerplate for hosting a typical Laravel application serverless in the AWS Cloud. 
We have combined the serverless.com framework, bref AWS Lambda and some AWS Cloudformation scripts to accomplish this. 
All AWS resources have been written as Infrastructure as Code. They are being used natively without the need to access passwords or secrets by hand. 
BTW: If you'd like to know why we're not betting on Tyler Otwell's Vapor to accomplish what we can, read on [here](https://dev.tech.exporo.com/blog/sls-laravel-agility/).. 

All resources are defined as a Cloudformation template in the serverless.yml file:
```yml
 environment:
    APP_KEY: !Sub '{{resolve:secretsmanager:${self:custom.UUID}-APP__KEY}}'
    APP_STORAGE: '/tmp'
    DB_HOST:
      Fn::GetAtt: [AuroraRDSCluster, Endpoint.Address]
    DB_PASSWORD: !Sub '{{resolve:secretsmanager:exporo-sls-laravel-dev-DB__PASSWORD}}'
    LOG_CHANNEL: stderr
    SQS_REGION: ${self:provider.region}
    VIEW_COMPILED_PATH: /tmp/storage/framework/views
    CACHE_DRIVER: dynamodb
    SESSION_DRIVER: dynamodb
    QUEUE_CONNECTION: sqs
    SQS_QUEUE: !Ref SQSQueue
    DYNAMODB_CACHE_TABLE: !Ref DynamoDB
    FILESYSTEM_DRIVER: s3
    AWS_BUCKET: !Ref S3Bucket
```

* AWS DynamoDB as  a Session driver
* AWS DynamoDB as a Cache driver
* AWS RDS Aurora serverless MySQL 5.6 as a Database
* AWS S3 as a Storage provider
* AWS Lambda event for triggering the cron jobs
* AWS SQS + Lambda Event for queueing processes

All resources have been paid for in a pay-as-you-go model. 

Since all resources are located in private subnets and hosted in a VPC, an EC2 instance is placed in a public subnet as a bastion host and NAT instance.
The NAT instance replaces a NAT gateway (~ 40€/month) with which Lambda functions can access the Internet. 
The instance type is t2.nano and costs about 5€ per month. 

Some load tests around RDS Aurora Serverless ACUs sizes can be found [here](https://dev.tech.exporo.com/blog/serverless-laravel-rds-serverless-benchmark/).

In some places this project is still a bit raw, because it is still quite new, so **feel free to contribute!**
 

## Requirements
<a name="requirements"/>

* Node.js 6.x or later
* PHP 7.2 or later
* AWS CLI
* Serverless Framework
* An AWS Account 

## Installation
<a name="installation"/>

```console
exporo_sls:~$ aws configure   
exporo_sls:~$ npm install -g serverless   
exporo_sls:~$ npm install  
exporo_sls:~$ composer install   
exporo_sls:~$ application/composer install  
```

## Deployment
<a name="deployment"/>

**Deployment**
```console
exporo_sls:~$ php artisan config:clear
exporo_sls:~$ serverless deploy --stage {stage} --aws-profile default
```

**Deployment chain**  
A serverless plugin *./serverless_plugins/deploy-chain.js* automatically creates an EC2 key pair and stores it in the AWS Parameter Store.
After deployment, the following steps were performed:
```console
exporo_sls:~$ serverless invoke -f artisan --data '{"cli":"migrate --force"}' --stage {stage} --aws-profile {profile}
exporo_sls:~$ aws s3 sync ./application/public s3://${service-name}-${stage}-assets --delete --acl public-read --profile {profile}
```

**Local database access**  
The same plugin fetches and displays all necessary parameters to access the database: 
```console
exporo_sls:~$ serverless ssh --stage {stage} --aws-profile default
```
Output:
```console
Serverless: -----------------------------
Serverless: -- SSH Credentials
Serverless: -----------------------------
Serverless: ssh ec2-user@18.185.33.123 -i ~/.ssh/exporo-sls-laravel-nat-instance
Serverless: MySql HOST: exporo-sls-laravel-nat-instance-aurorardscluster-scl4vnp4lyet.cluster-cadypvf3voom.eu-central-1.rds.amazonaws.com
Serverless: MySql Username: forge
Serverless: MySql Password: &a%<40I)ln]oo>F7Q]jUG!3OsVb2vM
Serverless: MySql Database: forge
```



## Assets
<a name="assets"/>

In addition to the private S3 bucket for the Laravel storage, a public bucket is created for the assets.

Assets should be used in the views like this:
```php
<img width="400px" src="{{ asset('exporo-tech.png') }}">
```

In the deployment chain, the S3 bucket should be synchronized with the public folder.
```shell
aws s3 sync public s3://${service-name}-${stage}-assets --delete --acl public-read --profile default
```

**local environment**  
Another docker container, for the delivery of the assets, with the address localhost:8080 will be built for the local environment. 

## Local development
<a name="local"/>

```console
exporo_sls:~$ docker-compose up -d
exporo_sls:~$ docker-compose exec php bash
exporo_sls:~$ open http://localhost
bash-4.2# cd /var/task/application/
bash-4.2# php artisan XYZ
```

## Demo application
<a name="demo"/>

The demo application implements various page counters using different AWS techniques including DB, Cache and Filesystem to store hits. 
The home controller only reads hits from resources and triggers an event that stores the hits asynchronously using as SQS queue. 
A cron job resets all page counters hourly. 

## Migrate your application
<a name="migration"/>

Empty the application folder, and insert your Laravel application. 
Almost all configurations are done in serverless.yml, but you will need to make a few minor changes to your Laravel application. 

##### 1: Add composer dependencies to your project

```console
exporo_sls:~$ application/composer require league/flysystem-aws-s3-v3
exporo_sls:~$ application/composer require bref/bref "^0.5"
```

##### 2: Removing error-causing env variables
Replace key and secret env vars with '' in:
- dynamodb in config/cache.php
- sqs in config/queue.ph
- s3 in config/filesystems.php


For example dynamodb in config/cache.php:
```php
'dynamodb' => [
            'driver' => 'dynamodb',
            'key' => '',
            'secret' => '',
            'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
            'table' => env('DYNAMODB_CACHE_TABLE', 'cache'),
            'endpoint' => env('DYNAMODB_ENDPOINT'),
        ],
```

##### 3: Update local filesystem
Update the root folder in **config/filesystem.php** to:

```php
'disks' => [

        'local' => [
            'driver' => 'local',
            'root' => env('APP_STORAGE', storage_path('app')),
        ],
```

##### 4: Set storage directory and create a temporary directory
Add this to the boot method in **app/Providers/AppServiceProvider.php**:

```php
app()->useStoragePath(env('APP_STORAGE', $this->app->storagePath()));

if (! is_dir(config('view.compiled'))) {
    mkdir(config('view.compiled'), 0755, true);
}
```

##### 5: Example application/.env
```
APP_KEY=base64:c3SzeMQZZHPT+eLQH6BnpDhw/uKH2N5zgM2x2a8qpcA=
APP_ENV=dev
APP_DEBUG=true

LOG_CHANNEL=stderr
APP_STORAGE=/tmp
CACHE_DRIVER=redis
SESSION_DRIVER=redis
REDIS_HOST=redis
VIEW_COMPILED_PATH=/tmp/storage/framework/views

DB_HOST=mysql
DB_USERNAME=homestead
DB_PASSWORD=secret
DB_DATABASE=forge

ASSET_URL=http://localhost:8080
```

## Todo
<a name="todo"/>

- add queue error / retry  handling
- display stderr from scheduled commands 

## CI/CD
<a name="CICD"/>

## Credits
<a name="credits"/>
  
* A big thanks is due to Tyler Otwell, whose framework Laravel and [Vapor](https://vapor.laravel.com/) service allows us to let our technological creativity run free.    
* The creators of the php Lambda layers called  [bref](https://bref.sh/docs/)
* The post [Migration Guide: Serverless + Bref + Laravel](https://medium.com/no-deploys-on-friday/migration-guide-serverless-bref-laravel-fbb513b4c54b) by Thiago Marini