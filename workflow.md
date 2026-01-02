Step 1: Building flask app
-
1. folder structure:
      ```
      aws-autoscaling-cicd-app/
      │
      ├── app/
      │   ├── app.py
      │   └── requirements.txt
      │
      ├── docker/
      │   └── Dockerfile
      │
      ├── terraform/
      │   ├── main.tf
      │   ├── variables.tf
      │   └── outputs.tf
      │
      ├── Jenkinsfile
      ├── .env
      └── README.md
      
      ```

How app works (local testing of /cpu and /health)
-> Run app on localhost
-> Open taskbar -> CPU utilization -> note down CPU utilization
-> set CPU_STRESS_TIME in seconds
-> hit localhost/cpu
-> note down CPU utilization for 'CPU_STRESS_TIME + 10' seconds  ->   it is spiked up

In production server, sometimes CPU utilization is spiked up for some reason. In this situation, ASG scales out. 
To test this functionality locally, /cpu creates this CPU utilization spike and in this situation it is checked whether ASG and ALB works as intended, that ASG creates new instances and 
ALB routes traffic to both of them

- Before hitting /cpu:
<img width="964" height="938" alt="image" src="https://github.com/user-attachments/assets/3dd330f7-d68b-46fe-8296-8d95289df16a" />

- After hitting /cpu:
<img width="939" height="961" alt="image" src="https://github.com/user-attachments/assets/93be70f0-c839-47b1-9230-af818a53ffa9" />

2. Idea: you create your app and you add 2 more endpoints to your app /health and /cpu

  - normally /health gives "UP" that means instance on which the app is running is in healthy state
  - /cpu causes CPU utilization spike

  - ALB (Application Load Balancer) hits /health every few seconds 
        - if gets HTTP 200  ->  instance stays in service
        - if fails    ->   instance is marked unhealthy  and ASG (Auto Scaling Group) replaces it.
  - /cpu increases CPU utilization 
  - CloudWatch notices that 'CPUUtilization > threshold'  -> ASG adds new instance (does not kill old one)

  - Scaling VS Replacement
    
    /health fails  ->   instance is replaced
    
    CPU high       ->   instance scaled out (add more)
    
    CPU normal again -> extra instances terminated later

Note: ASG creates instances, ALB just routes traffic to them.

2. Dockerize and test locally  ->  prereq: start docker desktop
   ```
    docker build -f docker-Dockerfile -t asg-alb:v1 .
   ```

   ```
   docker run -p 5000:5000 --name asg-alb-container asg-alb:v1
   ```
   Check :  ``` docker images``` and ```docker ps```

3. Create ECR repo and push docker image.
   1. create ecr repo
   - prereq
     ```
     aws --version
     aws sts -get-caller-identity
     ```
     if not then configure aws in terminal
     
     from the output, copy ```repositoryUri: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/autoscaling-flask-app```

   2. login docker to ECR :  this allow docker to push to aws
      ```
      $token = aws ecr get-login-password --region region_name
      docker login --username AWS --password $token ECR_repo_URI
      ```
      - aws ecr get-login-password -> creates temporary auth token
      - stored in token
      - docker logs in to ECR with that token
      - just update region and repo URI
      - this method of login exposes token in multiple places. (Never use in production, use IAM role instead). 

    3. Tag the image
       ```
            docker tag image_local_name:v1 ECR_REPO_URI:v1
       ```

    4. push image tp ecr
       ```
       docker push ECR_repo_URI:v1
       ```

    5. verify push
       ```
        aws ecr list-images --repository_name repo_name --region region_name
       ```
       just repo name, not repo URI
       you'll get your image tag

       now your app image is on ecr and it can be pulled
  
4. terraform networking resource provisioning
   1. create tf folder structure
      - vpc: vpc is a private IP space dedicated for you only.
        
        10.0.0.0/16 means your vpc has ip range from 10.0.0.0 to 10.0.255.255 containing 65536 IPs.

        in that there are 2 subnets : subnets means small part of vpc -> public subnet 1 and public subnet 2 -> usually these subnets are created in different availability zones for high availability. 

        10.0.1.0/24 containing 256 IPs and 10.0.2.0/24 containing 256 IPs

      - why do we need 2 subnets -> for high availability

        ALB needs atleast 2 subnets in different AZs (for routing traffic) and ASG needs diversity meaning if AZ1 (instance1) is down then what's the point in creating instance2 in same AZ4

      - ALB (application load balancer)
        ALB has public IPs and DNS names

      - internet gateway (IGW) : connects vpc to internet
     
      - route table : routes traffic to its destination -> 0.0.0.0/0 means anywhere on internet
     
      - route table association : it associates subnets to route table
     
      - target group : logical list of backends that a load balancer can send traffic to. (ec2 instances, lambda functions)
     
        target group hits '/health' and marks instance healthy or unhealthy and ALB trusts target group
        
        ALB never tallks to instance directly  :   ALB  ->  target group   ->   instance

      - Need of target group
        
        ALB would not know which ec2 instances exist
        
        health check is possible
        
        auto scaling integration is possible
        
        ASG automatically registers new instances and deregisters terminated instances into target group
        
      - traffic flow : browser -> public internet -> ALB -> target group ->  instance  ->   app
     
      - ASG is a controller not a server. It monitors CPU and based on it creates or destroys instances

        ASG is connected to 3 things :
        
        1. subnets (where instances need to be created)
        2. launch template (defines ami, instance type, security groups, iam role)
        3. Target group (where traffic goes)
        
        ASG is NOT inside VPC
        
        ASG does NOT receive traffic
        
        ASG does NOT expose IPs
   2. terraform init, plan and apply
  
5. create ALB, target group, listener:
   1. output alb dns name  ->  used to test "/health" and "/cpu"
   2. output target group arn  ->   later used by asg
  
6. create iam role for ec2

7. launch template


      















