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
   

















