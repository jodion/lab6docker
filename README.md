___
## Challenge Task-Cloud9: Running your image on Fargate

The work of Developers normally ends by sending the container image to the repository. However, to see the full picture, you will be deploying your container image on a Fargate cluster. You will first create a Cluster, create a Task definition of your container and create a service that will launch your Task on your Cluster.

1. Create an ECS cluster by running the following command:

```
aws ecs create-cluster --cluster-name fargate-cluster
```

The beginning of the ouput of the command should look similar to the following:

```json
{
    "cluster": {
        "status": "ACTIVE",
        "statistics": [],
        "clusterName": "fargate-cluster",
...
```

2. To create a task definition JSON file, you will use the Cloud9 IDE. In the Cloud9 menu bar, click **File -> New File** and enter the following into the new **Untitled1** tab that just opened.
        
```json
{
    "executionRoleArn": "change-me_FargateTaskRoleARN",
    "family": "webapp", 
    "networkMode": "awsvpc", 
    "containerDefinitions": [
        {
            "name": "webapp", 
            "image": "change_me-repositoryUri", 
            "portMappings": [
                {
                    "containerPort": 80, 
                    "hostPort": 80, 
                    "protocol": "tcp"
                }
            ]
        }
    ],
    "cpu": "256", 
    "memory": "512"
}
```
        

   - Go to the bottom of the **Connection Details** section in Qwiklabs. Copy the **FargateTaskRoleARN** to the clipboard and replace the entry **change-me_FargateTaskRoleARN** in the file you just inserted with the value from Qwiklabs.

   - Replace **change_me-repositoryUri** in the file you just inserted with the **repositoryUri** you noted in a previous step.

   - To save the file, click **File -> Save**. For **Filename**, enter: `webapptask.json` and click **Save**.

This is an example of a Task definition which does the following:

   - Specifies a Role ARN to allow the Task in ECS to download your image from your ECR repository.
   - Specifies the name of your container.
   - Specifies the URL of your image in your ECR repository.
   - Exposes tcp/80 to allow HTTP connections inbound.
   - Specifies 0.25vCPU and 512MB of RAM as resources for your Task.

3. Now that you have the definition in JSON created, you can create the Task Definition in ECS using it as an input by running the following command:

```
aws ecs register-task-definition --cli-input-json file://webapptask.json
```

The beginning of the ouput of the command should look similar to the following:

```json
{
    "taskDefinition": {
        "status": "ACTIVE",
        "networkMode": "awsvpc",
        "family": "webapp",
...
```

4. With the Task Definition created, you will now create the ECS Service for Fargate to launch a Task based on your Task Definition by executing the following command. There are two parameters to replace in the command. Make sure to only change those and not remove any brackets.

   - Replace **change-me_FargateSubnetID** with the **FargateSubnetID** field you can find at the bottom of the **Connection Details** section in Qwiklabs. This will tell Fargate in which subnet to place the Task.
   - Replace **change-me_FargateSecurityGroup** with the **FargateSecurityGroup** field you can find at the bottom of the **Connection Details** section in Qwiklabs. This will tell Fargate which security group to apply to the Task.

```
aws ecs create-service --cluster fargate-cluster --service-name webapp --task-definition webapp:1 --desired-count 1 --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[change-me_FargateSubnetID],securityGroups=[change-me_FargateSecurityGroup],assignPublicIp=ENABLED}"
```

The arguments for the create-service command are the following:
   - cluster: Name of the Fargate Cluster you created earlier
   - service-name: The name of the Service. It doesn't have to be the same name as the Task name
   - task-definition: The name of the Task Definition created in the previous step. The "1" represents the revision. You only uploaded one revision.
   - launch-type: Fargate to launch in Fargate, else it would be ECS. However, we don't want to manage the underlying EC2 instances so Fargate is the right answer here.
   - network-configuration: Which Subnet(s) to place the Task in, which Security Group(s) should be associated with the Task, assign a Public IP address to the Task

The beginning of the ouput of the command should look similar to the following:

```json
{
    "service": {
        "status": "ACTIVE",
        "taskDefinition": "arn:aws:ecs:us-east-1:012345678901:task-definition/webapp:1",
...
```

5. It is now time to test your deployment. The Task deployed doesn't have a load balancer in front of it and has a public IP address directly attached. To find it, you can either run a command or go through the AWS Management Console.

   - Using the AWS Management Console, on the Services menu, click **Elastic Container Service**, click the **fargate-cluster** link, click the **Tasks** tab and click on the link of the ID of your task under the **Task** column. You will find the **Public IP** address in the *Network* section.
   - Using the CLI, run the following command. Because there isn't a single command that returns that Public IP address, you need to execute 3 commands. One to find the Tasks ARN of your cluster, one to describe that Task ARN to find the Elastic Network Interface ID and one to describe the Elastic Network Interface ID to find the Public IP attached to it. Instead of running 3 commands, it makes makes use of *xargs* and *piping* to pass the output of one command into the other as well as the *query* argument to the AWS CLI to parse the output.

```
aws ecs list-tasks --cluster fargate-cluster --query taskArns --output text | xargs -I {} aws ecs describe-tasks --cluster fargate-cluster --tasks {} --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' --output text | xargs -I {} aws ec2 describe-network-interfaces --network-interface-ids {} --query NetworkInterfaces[0].Association.PublicIp --output text
```

6. The Task will take less than 1 minute to change state to RUNNING. The image needs to be downloaded from your ECR Repository which can take a few more seconds. You can run the following command to look at the last status of your Task:

```
aws ecs list-tasks --cluster fargate-cluster --query taskArns --output text | xargs -I {} aws ecs describe-tasks --cluster fargate-cluster --tasks {} --query tasks[0].lastStatus --output text
```

The output of this last command will change from **PROVISIONING** to **PENDING** to **RUNNING**.

7. Now that you have the Public IP address, open a Browser to that IP address and you should see your application live running on Fargate!
