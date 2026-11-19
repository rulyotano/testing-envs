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

### Testing Orchestrator
- Process Created messages
- 
