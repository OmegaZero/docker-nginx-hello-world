# docker-nginx-hello-world
Single page docker nginx 


NGINX webserver that serves a simple page containing its hostname, IP address and port as wells as the request URI and the local time of the webserver.

The images are uploaded to Docker Hub -- https://hub.docker.com/r/dockerbogo/docker-nginx-hello-world/.

How to run:
```
$ docker run -p 8080:80 -d dockerbogo/docker-nginx-hello-world
```

Now, assuming we found out the IP address and the port that mapped to port 80 on the container, in a browser we can make a request to the webserver and get the page below: 

![hello_world](./hello_world.png)


Reference: [Docker & Kubernetes](http://bogotobogo.com/DevOps/Docker/Docker_Kubernetes.php)

### This repo has been modified to allow for pushing to Amazon AWS ECS.  In order to do this the following changes had to be implemented in AWS:
Reference: [Use IAM Roles to Connect Github Actions in AWS](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/)
1) IAM --> Access Management --> Identity Providers
   - Add Provider
   - Select OpenID Connect
   - Provider URL: https://token.actions.githubusercontent.com
   - Click "Get thumbprint"
   - Audience: sts.amazonaws.com
   - Click Add Provider
   - The ARN of the provider will be used to create the trust policy of the role
     
2) IAM Role
   A role must be created which uses a Custom Trust Policy. The Policy is as follows:
   ```
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::********:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:sub": "repo:OmegaZero/docker-nginx-hello-world:*"
                }
            }
        }
    ]
   }
   ```
   - The value of Federated is the ARN of the Identity Provider created in Step 1
   - Change the sub tag to the path of your repo
   - Give the Role the following Permissions:
     - AmazonEC2ContainerRegistryFullAccess
     - AmazonECS_FullAccess

3) ECR Repository
   - Can be left Private
   - Repository Name should be "docker-nginx-hello-world" (this is the suffix of the entire uri)
   - All other details can be left default
   - After ECR creation, copy the URI for the next step
  
4) ECS Task Definition
   - Create new task definition
   - Task family name "docker-nginx-hello-world"
   - Launch Type: AWS Fargate
   - Operating System: Linux/X86_64
   - CPU: 1 vCPU
   - Memory: 2GB (which is the minimum at the time of this writing)
   - Task Role: Leave "-" (unless you already have a ecsTaskExecutionRole)
   - Task Execution Role: "Create new role" (unless you already have a ecsTaskExecutionRole)
   - Container 1:
     - Name: docker-nginx-hello-world
     - Image URI: Paste the URI of the ECR created in the last step
     - Essential Container: Yes
     - Container Port: 80
     - Protocol: TCP
     - App Protocol: HTTP
     - Leave all other defaults and click Create
  - After the Task definition is created, go into it and click the JSON tab, then copy the JSON
  - Replace the contents of .aws/hello-world-td-revision1.json in this GitHub repo

5) ECS Cluster
   - Cluster Name: docker-nginx-hello-world
   - Infrastructure: AWS Fargate
   - Click Create

6) ECS Cluster Service
   - Go into the cluster created in the last step and go to Services Tab
   - Click Create
   - Computer options: Launch Type
     - Leave defaulta (Launch Type: FARGATE, Platform Version: Latest)
     - Application Type: Service
     - Family: docker-nginx-hello-world
     - Service Name: docker-nginx-hello-world
     - Service Type: Replica
     - Desired Tasks: 1
     - Networking:
       - Click which vpc you desire (default should be fine if you have not changed your vpcs)
       - Security Group: Create new security group
         - Security group name: docker-nginx-hello-world
         - Inbound:
           - Type: HTTP
           - Source: Anywhere
       - Public IP is enabled
       - Click Create

After this, the GitHub Action workflow just needs to be modified. The workflow provided in .github/workflows/aws.yml should work with the above settings, however the value 
```
role-to-assume: arn:aws:iam::590183971879:role/GitHub
```
needs to be changed to the ARN of the role created in Step 2 above
