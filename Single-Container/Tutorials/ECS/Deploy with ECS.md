# Deploying a container using AWS ECS.
## Steps:
1. First push to Docker hub (covered in the previous tutorial [Running container on AWS EC2 using shell](3.%20Running%20container%20on%20AWS.md))
2. On the ECS dashboard, click on the "Create Cluster". A cluster is a logical grouping of tasks or services. Choose the cluster type and click on "Create Cluster".
3. Create a Task. Task definition is a blueprint for the container. It includes the container image, CPU, memory, networking, and other settings.
4. Create a Service. Service is a configuration that allows you to run and maintain a specified number of instances of a task definition simultaneously in an ECS cluster. In this case this is just an example of running a simple app so the service wont do much, a service is useful when you have multiple tasks running in a cluster, and also maintaining the task if it fails or stop running.

* Network settings in task/service should have the same port as the container port(80).
* Network settings should include the security group for enabling inbound traffic - HTTP port 80. which will allow the container to be viewed in the browser.You can edit the security group in the EC2 dashboard, and save a custom rule with the following settings:
    - Type: HTTP
    - Protocol: TCP
    - Port range: 80
    - Source: Anywhere ipv4
* The task definition should have the container image URI from Docker hub.
* Task launch type should be Fargate. OS should be Linux/Arm64 or Linux/x86_64 depending on the image architecture. From my experience the Node alpine image (When built in mac M1) should be Linux/Arm64.

## Update container changes if needed:
- If you changed some code in the app and want it to be reflected in the container (that is linked to the task in ECS), you need to build the image again and push it to Docker hub.
- You might also need ECS to pull the latest image from Docker hub again. You can do this by going inside the task settings, and there is a deploy button(drop-down menu) where you can choose the "update service" option. If this does not work It is also possible to just create a new task revision and it will simply pull the latest URI version of the image.
