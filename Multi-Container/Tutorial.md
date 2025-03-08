# Deploy multi-container setup to AWS ECS.
## Before starting:
- Since we have now multiple containers (in this example backend-nodejs and database-mongodb), we will obviously need to have them linked inside the same network in order to communicate with each other. Now usually it is done thanks to docker-compose, but since we are deploying to AWS ECS, ECS will create its own network environment in the service we will create - and the tasks associated to this service will all enjoy the same network environment. So we will have to do it manually - by translating the settings we have in docker-compose to settings that would be configured on AWS.
- The docker-compose file has a service called 'mongodb' and inside the app.js file of the backend-nodejs service, we are connecting to the mongodb service - with the service name 'mongodb': "@mongodb:27017/course-goals?authSource=admin" . This was perfect while we used docker-compose. But in this scenario of using AWS ECS - we will have to change the name mongodb to localhost. Make it flexible by using an env variable, that we can change for local use and for AWS use.
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
* Accessing the application will not be through the public ip of the service because the service might relaunch and get a new ip. So we will use the load balancer's DNS name, for a stable access point.
* I encountered an issue that the service was not able to run the tasks correctly. what fixed the issue was to make sure the mongodb is up and running before the backend-nodejs container starts. So I added a retry connection logic in the app.js file of the backend-nodejs service.
so make sure in the future to:
- Configure container dependencies in ECS (in the task definition).
- Keep the retry logic in your application.