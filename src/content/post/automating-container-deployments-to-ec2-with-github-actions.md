---
title: "Automating Container Deployments to EC2"
description: "How we integrated the Extension Engine's EC2 Docker deployment into our GitHub Actions CI/CD pipeline using SSM and CloudFormation."
publishDate: "03 Nov 2024"
tags: ["devops", "aws", "github-actions", "docker", "ec2", "cicd"]
draft: false
pinned: false
---

Automated evidence collection is essential for meeting today's compliance and security requirements. At risk3sixty, we've built a tool within our FullCircle platform called the **Extension Engine** — it enables us and our clients to create and run scripts that automatically collect evidence and upload it directly to the platform.

To streamline updates, we integrated the Extension Engine's deployment into our GitHub Actions CI/CD pipeline. Here's the architecture, the challenges, and how we solved them.

## Architecture Overview

Our platform primarily uses Amazon ECS Fargate to run containerized services, providing a managed, serverless environment that minimizes infrastructure overhead.

The Extension Engine is a different story. It requires **Docker-in-Docker (DinD)** capabilities, which benefit from EC2's flexible access control for running and managing containers. So while the rest of the platform runs on Fargate, the Extension Engine runs directly on an EC2 instance — and that meant building a custom deployment pipeline for it.

## Step 1: Configuring EC2 with CloudFormation

We used AWS CloudFormation to provision the EC2 instance and bootstrap a `deploy.sh` script onto it via `UserData`. Here's an abbreviated version of the Launch Configuration:

```yaml
ExtensionEngineLaunchConfig:
  Type: AWS::AutoScaling::LaunchConfiguration
  Properties:
    UserData:
      Fn::Base64:
        Fn::Sub:
          - |
            #!/bin/bash
            mkdir /fullcircle 
            cd /fullcircle

            touch deploy.sh
            echo '#!/bin/bash' | tee -a deploy.sh
            echo 'TAG=${!1:-latest}' | tee -a deploy.sh

            echo 'cd /fullcircle' | tee -a deploy.sh
            echo 'sudo aws ecr get-login-password --region ${AWS::Region} | sudo docker login --username AWS --password-stdin ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com' | tee -a deploy.sh
            echo 'sudo docker pull ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/fullcircle-extension-engine:${!TAG}' | tee -a deploy.sh
            echo 'sudo docker tag ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/fullcircle-extension-engine:${!TAG} extension-engine:latest' | tee -a deploy.sh
            echo 'sudo docker-compose down' | tee -a deploy.sh
            echo 'sudo docker-compose up -d' | tee -a deploy.sh

            chmod +x deploy.sh
```

The `deploy.sh` script does two things:

1. **Pull the image** — authenticates with ECR and pulls the specified tag
2. **Restart the container** — runs `docker-compose down` then `up -d` with the new image

## Step 2: IAM Permissions for Secure Access

We configured an IAM role specifically for the Extension Engine instance, scoped to only what it needs:

```yaml
ExtensionEngineRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: !Sub ${AWS::Region}-fullCircle-${Stage}-ExtensionEngineRole
    AssumeRolePolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: ec2.amazonaws.com
          Action: 'sts:AssumeRole'
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
      - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      - arn:aws:iam::aws:policy/AmazonSSMPatchAssociation
    Policies:
      - PolicyName: 'ECSTaskExecutionAndMonitoring'
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: 'Allow'
              Action:
                - 'ecr:GetAuthorizationToken'
              Resource:
                - '*'
            - Effect: 'Allow'
              Action:
                - 'ecr:BatchGetImage'
                - 'ecr:InitiateLayerUpload'
                - 'ecr:UploadLayerPart'
                - 'ecr:CompleteLayerUpload'
                - 'ecr:BatchCheckLayerAvailability'
                - 'ecr:GetDownloadUrlForLayer'
                - 'ecr:PutImage'
              Resource:
                - !Sub arn:${AWS::Partition}:ecr:${AWS::Region}:${AWS::AccountId}:repository/${repository-name}
```

This grants:

- **ECR access** — pull images from the repository
- **SSM access** — allows GitHub Actions to execute commands on the instance via Systems Manager without needing SSH

## Step 3: GitHub Actions Deployment Job

The final piece is the `deploy-ec2` job in the pipeline. By this point, the image has already been built and pushed to ECR in an earlier step. This job handles the rollout:

```yaml
deploy-ec2:
  name: Deploy containers to EC2 instances
  runs-on: ubuntu-latest
  needs: [set-global-vars, build-and-push]
  strategy:
    matrix:
      service: ['extension-engine']
      region: ['us-east-1', 'us-east-2']
      exclude:
        # Only deploy 'prod' to us-east-2
        - region: ${{needs.set-global-vars.outputs.AWS_ENVIRONMENT == 'prod' && 'dummy' || 'us-east-2'}}

  steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::${AWS::AccountId}:role/github-deploy-EC2
        role-duration-seconds: 1800
        aws-region: ${{matrix.region}}

    - name: Login to EC2 instance and run deploy.sh
      id: login-ec2
      env:
        AWS_ENVIRONMENT: ${{ needs.set-global-vars.outputs.AWS_ENVIRONMENT }}
        IMAGE_TAG: ${{ needs.set-global-vars.outputs.IMAGE_TAG }}
      run: |
        INSTANCE_ID="$( aws ec2 describe-instances \
          --region ${{matrix.region}} \
          --filters \
            Name=instance-state-name,Values=running \
            Name=tag:Name,Values=fullCircle-${{ env.AWS_ENVIRONMENT }}-${{matrix.service}} \
          --query "Reservations[0].Instances[0].InstanceId" \
          | sed 's/^"\(.*\)"$/\1/' )"
        aws ssm send-command \
          --document-name 'AWS-RunShellScript' \
          --parameters commands="/fullcircle/deploy.sh ${{env.IMAGE_TAG}}" \
          --output text \
          --instance-ids "${INSTANCE_ID}"
```

A few things worth noting:

- **Least-privilege role** — the pipeline assumes a scoped deployment role rather than using static credentials. The Trust Relationship on that role needs to be configured to allow your GitHub Actions runner to assume it.
- **Matrix deployment** — the `matrix` configuration handles multi-region rollouts and is easy to extend with additional services or regions.
- **SSM instead of SSH** — the pipeline looks up the instance by tag, then triggers `deploy.sh` via SSM `send-command`. No bastion host, no key management.

## Takeaways

This setup integrates the Extension Engine's deployment into our existing CI/CD pipeline with no special networking requirements and minimal operational overhead. The pattern — CloudFormation bootstraps the instance, IAM scopes access, SSM triggers the deploy, GitHub Actions orchestrates it all — is a solid model for any specialized workload that needs to run on EC2 rather than a managed container service.

It keeps the Extension Engine current and secure, which is exactly what evidence collection infrastructure needs to be.
