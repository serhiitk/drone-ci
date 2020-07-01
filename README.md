# Drone-CI pipeline
Drone-CI pipeline builds an image from a _Dockerfile_, run image in the test environment (http://docker.example.com:10081), and push the image to the Nexus repository.

## Variables
Variables defines in `./vars/default_vars.yml`

    drone_host_ip: ...
    gitlab_host_ip: ...
    nexus_host_ip: ...
    ....
    DRONE_GITLAB_CLIENT_ID:
    DRONE_GITLAB_CLIENT_SECRET:
    DRONE_RPC_SECRET:
    ....
    and etc.

## Prerequisites
### Preparation GitLab-server
- create an admin user for Drone projects: `drone-admin`
- create project for Drone-CI: `drone-test`
- create a GitLab OAuth application **(User -> Setting -> Applications):**

      Name: Drone-CI
      Redirect URI: http://drone.example.com:10080/login
      Confidential: Yes
      Scopes: 1) api 2) read_user   

- copy Application ID and Secret to `./vars/default_vars.yml` :

      DRONE_GITLAB_CLIENT_ID: "Application ID"
      DRONE_GITLAB_CLIENT_SECRET: "Secret"

### Create a Shared Secret (to authenticate communication between runners and your central Drone server )
- either generate HASH or plaintext:

      $ openssl rand hex 16

- copy Drone Shared Secret to `./vars/default_vars.yml` : 

      DRONE_RPC_SECRET:   # plaintext or a HASH as well

### Preparation Nexus-server repository
- create repository: **drone-test-app1** (parameters: docker hosted, **https port - 18092**, enable Docker API)
- create Role with "admin" and "view" privileges to **drone-test-app1** repository (example: Drone-CI Role)
- create repository user: **droneci-user**. And grant Role created in the previous step 

### Preparation Docker server (if it's the first start)
- start deployment Docker server (from repository: **docker_systemd**): 

      $ absible-playbook deploy_containers.yml

- copy **private_key** to `./drone-ci/hosts/Docker-server/certs/` directory with **chmod 0600** use `cp -p` command

## Deploy Drone-CI Server and Runner (run in containers on Docker-server)
- start deployment Drone-CI (Drone admin user as on GitLab server - `drone-admin`): 

      $ absible-playbook deploy_drone.yml

- set **drone-test-app1** repository secrets on Drone-CI Server - **drone.example.com:10080 -> select project: `drone-test-app1` -> Settings -> Secrets:**

      DRONE_NEXUS_USER = droneci-user
      DRONE_NEXUS_PASS = dronepass123 

## Environment setup
- start `environment_setup.yml` to perform the next tasks:
     - to Docker server copy Nexus SSL-certificate for connection between Docker and Nexus servers. 
     - to Docker server resolve local DNS names (gitlab, nexus and drone servers) by adding hosts to `/etc/hosts` file.
     - to GitLab server resolve local DNS names (gitlab, nexus and drone servers) by adding hosts to `/etc/hosts` file.

           $ absible-playbook environment_setup.yml

## Configure pipeline variables in `.drone.yml`
- set your **local hosts** for pipeline containers in anchors: `&extra_gitlab_host` and `&extra_nexus_host` .
- set your **environment variables** for pipeline containers:
       
      CFG_GITLAB_URL_SCHEMA: "http"
      CFG_GITLAB_HOST_NAME: "gitlab.example.com"
      CFG_GITLAB_PROJECT_NAME: "drone-test"
      CFG_NEXUS_HOST_NAME: "nexus.example.com"
      CFG_NEXUS_REPOSITORY_PORT: 18092
      CFG_NEXUS_REPOSITORY_NAME: "drone-test-app1"
      CFG_TEST_CONTAINER_NAME: "drone-test-env-app1"

## Push test project from ./drone-ci/repository_files/ to the created repository on GitLab server
    $ git remote add origin http://gitlab.example.com/drone-admin/drone-test.git
    $ git push -u origin master