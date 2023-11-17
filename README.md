# Concourse Docker
Please clone this repo
```sh
git clone https://github.com/endijsbertans/concourse.git
```
Then change localhost to YOUR public IPV4
```sh
cd concourse
nano docker-compose.yml
```
![image](https://github.com/endijsbertans/concourse/assets/97877531/26d57c17-7b03-4282-bf64-d1514f2e5345)

you'll first need to execute
`./keys/generate` - this will generate credentials used to authorize the
Concourse components with each-other:

```sh
$ ./keys/generate
wrote private key to /keys/session_signing_key
wrote private key to /keys/tsa_host_key
wrote ssh public key to /keys/tsa_host_key.pub
wrote private key to /keys/worker_key
wrote ssh public key to /keys/worker_key.pub
```

Next, run `docker-compose up -d` to start Concourse in the background:

```sh
$ docker-compose up -d
Starting concourse-docker_db_1 ... done
Starting concourse-docker_web_1 ... done
Starting concourse-docker_worker_1 ... done
```

The default configuration sets up a `test` user with `test` as their password
and grants them access to `main` team.

If things seem to be going wrong, check the logs for any errors:

```sh
$ docker-compose logs -f
Attaching to concourse-docker_worker_1, concourse-docker_web_1, concourse-docker_db_1
...
```
WHOLAA! check if it woorks!
![image](https://github.com/endijsbertans/concourse/assets/97877531/f37233d5-06ad-46c7-a2dc-b84ead8babf0)

## Building `concourse/concourse`

The `Dockerfile` in this repo is built as part of our CI process - as such, it
depends on having a pre-built `linux-rc` available in the working directory, and
ends up being published as `concourse/concourse`.
Now let's install fly!
```sh
curl 'http://localhost:8080/api/v1/cli?arch=amd64&platform=linux' -o fly \
    && chmod +x ./fly && mv ./fly /usr/local/bin/
# now lets create a target called tutorial, change localhost to your ipv4
fly -t tutorial login -c http://localhost:8080 -u test -p test
```
Now create a hello world pipeline and test your work!
```yml
jobs:
- name: hello-world-job
  plan:
  - task: hello-world-task
    config:
      # Tells Concourse which type of worker this task should run on
      platform: linux
      # This is one way of telling Concourse which container image to use for a
      # task. We'll explain this more when talking about resources
      image_resource:
        type: registry-image
        source:
          repository: busybox # images are pulled from docker hub by default
      # The command Concourse will run inside the container
      # echo "Hello world!"
      run:
        path: echo
        args: ["Hello world!"]
```
Run the pipeline with 
```sh
$ fly -t tutorial set-pipeline -p hello-world -c hello-world.yml
# pipelines are paused when first created
$ fly -t tutorial unpause-pipeline -p hello-world
# trigger the job and watch it run to completion
$ fly -t tutorial trigger-job --job hello-world/hello-world-job --watch
```
Also some nice commands for troubleshooting <3
```sh
fly -t <target> builds -j <pipeline>/<job>
fly -t <target> hijack -j <pipeline>/<job> --build <build-id>
fly -t <target> intercept -j <pipeline>/<job> --build <build-id>
fly -t <target> workers
fly targets
```


Also here is a little more complicated pipeline
```
esource_types:
- name: sonar-runner
  type: docker-image
  source:
    repository: cathive/concourse-sonarqube-resource
    tag: latest

resources:
- name: html-project
  type: git
  source:
    uri: https://github.com/endijsbertans/BChtml.git
- name: code-analysis
  type: sonar-runner
  source:
    host_url: http://172.31.40.56:9000
    login: sqp_4c954457f81d4f9384a147315948761ddc094d65

jobs:
- name: deploy-nginx
  plan:
  - get: html-project
    trigger: true
  - task: deploy-nginx
    config:
      platform: linux
      image_resource:
        type: registry-image
        source: {repository: ubuntu}
      inputs:
      - name: html-project
      outputs:
      - name: sonarqube-analysis-input
      run:
        path: /bin/bash
        args:
        - -c
        - |
          cp -r html-project/* sonarqube-analysis-input
          export DEBIAN_FRONTEND=noninteractive
          apt-get update
          apt-get install -y python3 python3-dev
          apt install -y python3.11-venv
          # Set DEBIAN_FRONTEND to noninteractive to avoid prompts during Ansible installation
          apt install curl -y
          echo "ansible executed"
          apt install -y curl
          apt install -y unzip
          curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
          unzip awscliv2.zip
          ./aws/install
          curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
          dpkg -i session-manager-plugin.deb
          echo "SSM installde"
          aws configure set region eu-west-1
          aws configure list
          echo "region set trying to connect"
          aws ssm send-command --instance-ids  i-0518c0bccc7f5c687 --document-name "AWS-RunShellScript" --parameters commands="sudo docker stop nginx; sudo docker rm nginx; sudo docker run -d -p 80:80 --name n>

          echo "test"

  - in_parallel:
    - put: code-analysis
      params:
        project_path: sonarqube-analysis-input
        project_key: p2
        sources: ["."]


```
![image](https://github.com/endijsbertans/concourse/assets/97877531/880cf87b-2c43-4554-a3ab-9cff011355d5)

