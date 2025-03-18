# Deploy multi-container setup to AWS ECS.
## Before starting:
- Since we have now multiple containers (in this example backend-nodejs and database-mongodb), we will obviously need to have them linked inside the same network in order to communicate with each other. Now usually it is done thanks to docker-compose, but since we are deploying to AWS ECS, ECS will create its own network environment in the service we will create - and the tasks associated to this service will all enjoy the same network environment. So we will have to do it manually - by translating the settings we have in docker-compose to settings that would be configured on AWS.
- The docker-compose file has a service called 'mongodb' and inside the app.js file of the backend-nodejs service, we are connecting to the mongodb service - with the service name 'mongodb': "@mongodb:27017/course-goals?authSource=admin" . This was perfect while we used docker-compose. But in this scenario of using AWS ECS - we will have to change the name mongodb to localhost. Make it flexible by using an env variable, that we can change for local use and for AWS use.
- We will use a load balancer for managing traffic in an efficient way, and also for health check to the route /goals. and also for a stable access point to the application (since the service might relaunch and get a new ip).with the load balancer's DNS name.
## Steps:
## Container 1 (backend-nodejs):
1. run docker build on the backend folder. (add ENV variables to the dockerfile if needed) 
2. Push to docker hub. 
3. Create a AWS Fargate Cluster.
4. Create a Task Definition: 
- choose AWS Fargate
- OS type: linux/arm64
- Task role: choose from drop-down option (only 1 option available).
- Add the env variables from the local env file(backend.env). (entered as key & value).
* if running locally we might need different env values like the MONGODB_URL=mongodb has mongodb as the value, but in AWS we will have to change it to localhost. so copy the key 'MONGODB_URL' but make sure the value is 'localhost'. localhost is the default machine & network all containers will sit in(taking in account they are in the same task), on AWS. 
- Under Docker configuration -> command -> type: node,app.js (This simply overwrites the CMD in the Dockerfile that has a different command to be started with the nodemon for development purposes). 
5. Click 'add container' to continue to adding the 2nd container. 
## Container 2 (database-mongodb):
1. type in the name of the container, and in the imageURI just type 'mongo' (since we are using the official mongo image).
2. Port Mapping: Add the default port of mongo which is 27017.
3. Add the env variables from the local env file(mongo.env).
4. later we might need to add a volume to persist the data.
5. Go back to the cluster, and click on 'create service'.
6. Inside the service configuration:
- Launch type: FARGATE
- Task Definition: choose the task definition we just created.
- load balancer: add an application type load balancer and configure the health check path to /goals. (the default is / which is the root path, but we want to check if the backend is up and running, so we will use /goals and there is nothing configured to / so it will report an error an relaunch itself over and over).
* Accessing the application will not be through the public ip of the service because the service might relaunch and get a new ip. So we will use the load balancer's DNS name, for a stable access point. That DNS address will be shown when the service is up and running - you can access it inside the service details -> confifuration and networking -> and under Network Configuration -> copy the link that is shown under "DNS names". now dont forget to add at the end of the link the /goals path to access the backend db. in postman get/post requests.
* I encountered an issue that the service was not able to run the tasks correctly. what fixed the issue was to make sure the mongodb is up and running before the backend-nodejs container starts. So I added a retry connection logic in the app.js file of the backend-nodejs service.
so make sure in the future to:
- Configure container dependencies in ECS (in the task definition).
- Keep the retry logic in your application.
* * Note that all data saved in the mongodb database will be lost when the container is stopped/restarted. So we will need to add a volume to persist the data.
7. Add persistent storage to the mongodb container:
- Create a new security group policy that will be used by the EFS (Elastic File System) that we will create.
- Navigate (from search) to EC2 , then to Security Groups -> Create security group -> Add a name (like efs-sc), Add an inbound rule: type: NFS, protocol: TCP, port range: 2049, source: Custom -> and then next to it choose from dropdown the security group that is associated with the containers.(This will essentially allow the containers to "talk" to the EFS).
- Search for EFS (Elastic File System) in the AWS console -> Create a new file system 
- click next on the first page (nothing to change), and on the second page on "Network access" -> add the security group we just created to ALL of the availability zones(subnets) that are available. then next on all the next pages until the create button.
- Go to the task definition, create new task revision (or could have been done while creating the task definition).
- Scroll down Storage -> Add volume -> Choose EFS (Elastic File System) -> in file system ID choose the EFS we just created. (You can choose to add access point or not, it is for targeting a nested path inside the file system, but here we will just use the entire system as a volume. but again it is useful if i have multiple volumes and i dont want to create for them multiple filesystems).
- Now just below there is a "Add mount point" button, click it and add the mount point to the mongodb container. (this will mount the EFS to the container).in the container path type: /data/db (this is the default path where mongo stores its data and also what was originally configured in the docker-compose file as a bindmount).
## You can now see that data presists even after stopping and restarting the container, you can give it a quick test in postman, go to the POST request with the DNS address, and click body -> raw -> JSON -> and type in the body: {"text":"test"} and send the request. then go to the GET request, you should see the data you just posted. And if you stop the container and start it again,You can see the data will still be there even if you do the "force new deployment" on the service. 
* * Note that if I do a force new deployment when updating the service, I might encounter a collision scenario where the old task is still running and occupying the EFS, so the new task will not be able to mount the EFS therefore it will be stuck and will not complete. So I will have to stop the old task manually - click on stop all tasks to make sure.(this is because the way updating and forcing a new task works, it will not stop the old task until the new task is up and running and healthy).