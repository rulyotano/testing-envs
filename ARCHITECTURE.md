## Functional Requirements
- Web application that user logs in.
- User can see all created testing envs
- User can see own created testing envs (only user & admin has update permissions)
- User can create a new testing environment
- User can update created testing envs (if has permission)
- User can delete a testing env (if has permissions)

## Environment Creation
- User starts create of new testing env.
- User pick a name (only letters and numbers)
- User pick the services to use and the branch.
- User complete the creation.

## Creation under the hook
- The a new route will be created, for example: "createdtestingenvname.testings.company-url.com"
- If the service has databases configured (script files, db-type) create the databases, for example, mongo-db: createdtestingenvname-features-db created from script file
- If already there is a image built for the latest branch tag use it, if not build an image, push it to CR, save map commit-id:image-tag
- Copy terraform files from services testing env templates, into argo-cd directory.
- Set service status as Created. Needs to wait for Argo to complete the deployment.

## Components identified

### TestingEnvs Frontend
- SPA application
- Lets use VUE js
- Main views
  - Login
  - List all envs created
  - Filter to my envs
  - Create new testing env
  - Testing env details
  - User top menu
  - Left navigation menu
  - Admin:
    - Crud for available services

### TestingEnvs API
- Lets use Golang
- Database 
  - MongoDb
- Configuration
  - Available services for the testings
    - Repository url
    - Name
    - (all the rest of config can be here or in each repo, preferred each repo)
  - Available branches for a service (query to git server)
- Functionalities
  - Create testing env (mongo collection)
    - One record with the service name + other metadata
    - Status
    - List of services
      - status of each service
    - Initial state = created
    - Publish message in RabbitQ (testing created)
  - Delete testing env
  - Listen testing status change

### Task Processor
- Process the messages.
- For example:
  - Process the message "testing created".
    - check the type "testing created"
    - find the right processor class e.g. "CreateTestHandler"
    - Execute it with the arguments from the message
      - Inside the handler:
        - Create the K8s terraform with the container namespace and common stuff for the new testing env
        - For each service and branch
          - Check if the docker image was already created for the latest branch commit. If not:
            - Set state to building-image (it can be a sync http request)
            - Start the build image action/pipeline (on finish should update db)
            - (wait for the image is built, can use a CRON to check status of all building-image state testing services...)
          - If image already created, just fullfil the service template (each service should have a testing-env terraform template in .argocd or .devops directory) and update status to Done.

### Container Registry service
- To store the images, just use a free one just to POC

### Git repository service
- Free one just for POC. GitLab maybe. If has Github Action like is a plus.

### ArgoCd

### DBs
- For POC lets have Mongo + Postgres

### RabbitMq
- Queues for the testings envs
- Dynamic queues created for envs that needed.

### Repositories (pre existing)
- argo-cd-devops: to save base terraform files for argocd (missing k8s package like library)
- mock services.
  - each service should have a .devops directory with each cluster. terraform files. eg. /prod-latam /prod-emma /test ... and base template configs. Test config variables and config should have placeholders for testing-name.
  - rabbit config (is enabled or not, etc)
  - mongo config (is enabled or not, initial seed)
  - postgres config (is enabled or not, initial seed)




