# 과제 1번 솔루션

## Files for different deploy types
|     Type      | appspec file |    file needed |
| ------------- | ------------- | ------------- |
| `Pipeline with ECS Rolling Update`  |    `X`     | `imagedefinitions.json`|
| `Pipeline with ECS Blue Green Deployment`  |    `O`     | `imageDetail.json`|

## buildspec 
#### ECR로 푸쉬만
```
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - ACCOUNT_ID=$(aws sts get-caller-identity | jq ".Account" -r)
      - REPOSITORY_URI=$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/skills-ecr
      - aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com"
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:stress .
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:stress
```
#### 서비스 강제 업데이트
```
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - ACCOUNT_ID=$(aws sts get-caller-identity | jq ".Account" -r)
      - STRESS_URI=$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/skills-stress
      - PRODUCT_URI=$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/skills-product
      - aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com"
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t  $PRODUCT_URI:latest ./product/
      - docker build -t  $STRESS_URI:latest ./stress/
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $PRODUCT_URI:latest
      - docker push $STRESS_URI:latest
      - aws ecs update-service --cluster skills-ecs-cluster --service skills-product-svc --force-new-deployment
      - aws ecs update-service --cluster skills-ecs-cluster --service skills-stress-svc --force-new-deployment
```
#### buildspec with rolling update
```
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - ACCOUNT_ID=$(aws sts get-caller-identity | jq ".Account" -r)
      - REPOSITORY_URI=$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/skills-ecr
      - IMAGE_TAG=$(TZ=Asia/Seoul date +"%Y-%m-%d.%H.%M.%S") #한국시간
      - aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com"
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - "printf '[{\"name\": \"app\", \"imageUri\":\"%s\"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json"
      - echo "$(jq .containerDefinitions[].image=\"$REPOSITORY_URI:$IMAGE_TAG\" taskdef.json)" > taskdef.json
      # taskdef에 container가 2개이상 경우 .containerDefinitions에 []를 [0]로 수정
artifacts:
  files:
    - imagedefinitions.json
    - taskdef.json
    - appspec.yaml
```
#### buildspec with blue green deployment
```
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - ACCOUNT_ID=$(aws sts get-caller-identity | jq ".Account" -r)
      - REPOSITORY_URI=$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com/skills-ecr
      - IMAGE_TAG=$(TZ=Asia/Seoul date +"%Y-%m-%d.%H.%M.%S")
      - aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.ap-northeast-2.amazonaws.com"
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:$IMAGE_TAG .
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '{"ImageURI":"%s"}' $REPOSITORY_URI:$IMAGE_TAG > imageDetail.json
      - echo Writing image definition file...
      - echo "$(jq .containerDefinitions[].image=\"$REPOSITORY_URI:$IMAGE_TAG\" taskdef.json)" > taskdef.json
      # taskdef에 container가 2개이상 경우 .containerDefinitions에 []를 [0]로 수정
artifacts:
  files: 
    - imageDetail.json
    - taskdef.json
    - appspec.yaml
```

## appspec.yml
```
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "<IMAGE>" #수정할 필요없음
        LoadBalancerInfo: 
          ContainerName: "app"  # ECS taskdefinition에 설정했던 container 이름 추가
          ContainerPort: 8080
```

## taskdef.json
```
{
    "containerDefinitions": [
        {
            "name": "skills-product-ctn", # 이름 수정
            "image": "<IMAGE>", # 수정 필요없음
            "cpu": 512,
            "memory": 1024,
            "portMappings": [
                {
                    "name": "skills-product-port", # 설정했던 포트매핑 이름으로 변경
                    "containerPort": 8080,
                    "hostPort": 8080,
                    "protocol": "tcp",
                    "appProtocol": "http"
                }
            ],
            "essential": true,
            /* "dependsOn": [
                {
                    "containerName": "envoy",
                    "condition": "HEALTHY"
                }
            ], */ # appmesh를 필요할 경우
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-create-group": "true",
                    "awslogs-group": "/ecs/skills-product-td",
                    "awslogs-region": "ap-northeast-2",
                    "awslogs-stream-prefix": "ecs"
                }
            },
            "healthCheck": {
                "command": [
                    "CMD-SHELL",
                    "curl -fLs http://localhost:8080/health > /dev/null || exit 1"
                ],
                "interval": 5,
                "timeout": 2,
                "retries": 1,
                "startPeriod": 0
            }
        },
        /* {
            "name": "envoy",
            "image": "840364872350.dkr.ecr.ap-northeast-2.amazonaws.com/aws-appmesh-envoy:v1.26.4.0-prod",
            "cpu": 0,
            "memoryReservation": 128,
            "essential": true,
            "environment": [
                {
                    "name": "APPMESH_VIRTUAL_NODE_NAME",
                    "value": "mesh/skills-mesh/virtualNode/product-service-vn"
                },
                {
                    "name": "ENABLE_ENVOY_XRAY_TRACING",
                    "value": "1"
                },
                {
                    "name": "ENABLE_ENVOY_STATS_TAGS",
                    "value": "1"
                }
            ],
            "user": "1337",
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/microservice-demo",
                    "awslogs-region": "ap-northeast-2",
                    "awslogs-stream-prefix": "skills-product/envoy"
                }
            },
            "healthCheck": {
                "command": [
                    "CMD-SHELL",
                    "curl -s http://localhost:9901/server_info | grep state | grep -q LIVE"
                ],
                "interval": 5,
                "timeout": 2,
                "retries": 1,
                "startPeriod": 0
            }
        }, */ appmesh 필요할 경우
        /* {
            "name": "xray-daemon",
            "image": "amazon/aws-xray-daemon",
            "cpu": 32,
            "memoryReservation": 256,
            "portMappings": [
                {
                    "containerPort": 2000,
                    "hostPort": 2000,
                    "protocol": "udp"
                }
            ],
            "essential": true,
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-create-group": "true",
                    "awslogs-group": "/ecs/skills-product-td",
                    "awslogs-region": "ap-northeast-2",
                    "awslogs-stream-prefix": "skills-product/xray"
                }
            }
        }
    ], */ xray 필요할 경우
    "family": "skills-product-td", # 수정 필요
    "taskRoleArn": "arn:aws:iam::148010479988:role/ecsTaskExecutionRole",
    "executionRoleArn": "arn:aws:iam::148010479988:role/ecsTaskExecutionRole",
    "networkMode": "awsvpc",
    "status": "ACTIVE",
    "requiresAttributes": [
        {
            "name": "ecs.capability.execution-role-awslogs"
        },
        {
            "name": "com.amazonaws.ecs.capability.ecr-auth"
        },
        {
            "name": "com.amazonaws.ecs.capability.docker-remote-api.1.17"
        },
        {
            "name": "com.amazonaws.ecs.capability.docker-remote-api.1.21"
        },
        {
            "name": "com.amazonaws.ecs.capability.task-iam-role"
        },
        {
            "name": "ecs.capability.aws-appmesh"
        },
        {
            "name": "ecs.capability.container-health-check"
        },
        {
            "name": "ecs.capability.execution-role-ecr-pull"
        },
        {
            "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
        },
        {
            "name": "ecs.capability.task-eni"
        },
        {
            "name": "com.amazonaws.ecs.capability.docker-remote-api.1.29"
        },
        {
            "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
        },
        {
            "name": "com.amazonaws.ecs.capability.docker-remote-api.1.19"
        },
        {
            "name": "ecs.capability.container-ordering"
        }
    ],
    "compatibilities": [
        "EC2",
        "FARGATE"
    ],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "1024",
    "memory": "2048",
    /* "proxyConfiguration": {
        "type": "APPMESH",
        "containerName": "envoy",
        "properties": [
            {
                "name": "ProxyIngressPort",
                "value": "15000"
            },
            {
                "name": "AppPorts",
                "value": "8080"
            },
            {
                "name": "EgressIgnoredIPs",
                "value": "169.254.170.2,169.254.169.254"
            },
            {
                "name": "IgnoredUID",
                "value": "1337"
            },
            {
                "name": "ProxyEgressPort",
                "value": "15001"
            }
        ]
    }, */ appmesh 필요할 경우
    "registeredBy": "arn:aws:iam::148010479988:root"
}
```

## Terraform으로 RDS & Secret 생성
```
resource "aws_rds_cluster" "db" {
  cluster_identifier          = "skills-rds-cluster" #수정
  database_name               = "product" # 수정
  availability_zones          = ["ap-northeast-2a", "ap-northeast-2b", "ap-northeast-2c"]
  db_subnet_group_name        = aws_db_subnet_group.db.name
  master_username             = "skills" #수정
  master_password             = "Skills2024**" # 수정
  vpc_security_group_ids      = [aws_security_group.db.id]
  skip_final_snapshot         = true
  storage_encrypted           = true
  engine                      = "aurora-mysql"
}

resource "aws_rds_cluster_instance" "db" {
  cluster_identifier     = aws_rds_cluster.db.id
  instance_class         = "db.r6g.large" # 수정
  identifier             = "skills-db-instance" # 수정
  engine                 = "aurora-mysql"
}

resource "aws_secretsmanager_secret" "db" {
  name_prefix = "product/dbcred" # 수정
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = jsonencode({
    "username" = "skills" # 수정
    "password" = "Skills2024**" # 수정
    "engine" =  "mysql"
    "host" = aws_rds_cluster.db.endpoint
    "port" = aws_rds_cluster.db.port
    "dbClusterIdentifier" = aws_rds_cluster.db.cluster_identifier
    "dbname" = aws_rds_cluster.db.database_name
  })
}
```
