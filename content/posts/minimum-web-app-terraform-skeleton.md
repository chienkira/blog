---
title: "Minimum Web App Terraform Skeleton (English)"
date: 2019-11-03T16:56:11+09:00
draft: false
tags: [terraform, tech, infra, aws]
language: en
toc: true
authors: [chienkira]
cover: /blog/images/terraform-x-aws-1.png
---

**I made a Skeleton for Terraform code. Basically I'm supposed to use it for future projects, but then just thought it might be useful for others so that want to share it with you. Also I tried to avoid having to copy and paste codes when deploye in multiple environments for same app. Hopefully, you will be able to reuse at least a part or an idea to help you with your work.**

## Directory layout
```bash
|--.gitignore
|--README.md
|--_resources
|  |--ec2.tf
|  |--network.tf
|  |--rds.tf
|  |--s3.tf
|  |--sg.tf
|  |--variables.tf
|--prod
|  |--main.tf
|--test
|  |--main.tf
```

## Check the source code here
https://github.com/chienkira/minimum-web-app-terraform-skeleton

---
> #### Minimum-web-app-terraform-skeleton
> Minimum Terraform skeleton (sample) for web app 
> 
> #### Features
> - All resources are defined as terraform module (D-R-Y infra code)
> - Allow you create same infra for both test and prod environments with only 1 command
> - Comes with minimum setup, you can add everything you need, also custom all settings
>     - This skeleton uses EC2 as web server, RDS as database server and S3 as file storage
> - Allocate EIP and assign static IP address to your EC2 instance so it's easier for you to setup domain stuffs after
> - Create a new VPC for your web app (good practise to not use the default VPC)
> 
> #### How to deploy
> ```bash
> $ # checkout this repository
> $ cd test                           # change to directory of enviroment you want to create (test or prod environment)
> $ terraform init && terraform plan  # check what resources will be create
> $ terraform apply                   # apply (deploy) all resources
> ```
> 
---

All comment and feedback are welcome.
