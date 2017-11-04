GitLab CI/CD
=======


DevOps course, practices with [Google Cloud Platform](https://cloud.google.com/).

## Homework 19

 - Create instance in GCE by `docker-machine` for [GitLab CI](https://about.gitlab.com). Change the environment variables for the Docker Client and connect to the remote Docker Engine
```bash
~$ docker-machine create --driver google \
--google-project <project_id> \
--google-zone europe-west1-b \
--google-machine-type n1-standard-1 \
--google-disk-size 100 \
--google-open-port 80 \
--google-open-port 443 \
--google-machine-image $(gcloud compute images list --filter ubuntu-1604-lts --uri) \
gitlab-ci

~$ eval $(docker-machine env <docker_instance_name>)
```

 - Need to install `docker-compose` on the remote vm
```bash
~$ docker-machine ssh <docker_instance_name>
gitlab-ci:~$ sudo -i
gitlab-ci:~# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
gitlab-ci:~# sudo apt-get update
gitlab-ci:~# apt-get install docker-compose
gitlab-ci:~# docker-compose --version
docker-compose version 1.8.0, build unknown
```

 - Prepare the environment for `docker-compose` and `docker-compose.yml`
 ```bash
root@gitlab-ci:~# mkdir -p /srv/gitlab/config /srv/gitlab/data /srv/gitlab/logs
root@gitlab-ci:~# vim /srv/gitlab/docker-compose.yml
```

```bash
web:
  image: 'gitlab/gitlab-ce:latest'
  restart: always
  hostname: 'gitlab.example.com'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'http://<YOUR-VM-IP>'
  ports:
    - '80:80'
    - '443:443'
    - '2222:22'
  volumes:
    - '/srv/gitlab/config:/etc/gitlab'
    - '/srv/gitlab/logs:/var/log/gitlab'
    - '/srv/gitlab/data:/var/opt/gitlab'
```

 - Run docker container by `docker-compose`
```bash
root@gitlab-ci:~# docker-compose -f /srv/gitlab/docker-compose.yml up -d
root@gitlab-ci:~# docker-compose -f /srv/gitlab/docker-compose.yml logs
```

 - Open http://<your-vm-ip> and create new administrative 'root' account
 - Authorized like 'root' and login in GitLab CI
 - Nobody should not sign up in GitLab except you. Let's disable this option in configuration: `Admin area` -> `Settings` -> `Sign-up Restrictions` -> `Sign-up enabled` = false -> `Save`

 - Any project is a part of some group, let's create a new group: `Groups` -> `Create group`
```bash
   Group path: http://<your-vm-ip>/homework
   Group name: homework
   Description: 'DevOps course, homework 19'
   Visibility Level: private
```

 - Create new project: `New project` -> `Create project`
```bash
   Project path: http://<your-vm-ip>/gitlab-ci-01
   Project name: gitlab-ci-01-homework
   Visibility Level: private
```

 - Clone your project `gitlab-ci-01` and make first commit
```bash
~$ git clone http://<your-vm-ip>/homework/gitlab-ci-01.git
~$ touch README.md
~$ git add README.md
~$ git status
~$ git commit -m "Add README"
~$ git push -u origin master
```

 - Let's continue to define CI/CD Pipeline for the project
```bash
~$ vim .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

build_job:
  stage: build
  script:
    - echo 'Building'

test_unit_job:
  stage: test
  script:
    - echo 'Unit Testing 1'

test_integration_job:
  stage: test
  script:
    - echo 'Integration Testing 2'

deploy_job:
  stage: deploy
  script:
    - echo 'Deploy'
```

```bash
~$ git add .gitlab-ci.yml
~$ git commit -m 'Add pipeline definition'
~$ git push origin master
```

 - Make and register runner for the project
   - `CI/CD` -> `Pipelines` = pipeline is ready to run
   - Settings -> `CI/CD` -> `Expand` on `Runners settings` = copy token for 'Specific Runners'
```bash
~$ docker-machine ssh <docker_instance_name>
~$ sudo -i
gitlab-ci:~# docker run -d --name gitlab-runner --restart always \
   -v /srv/gitlab-runner/config:/etc/gitlab-runner \
   -v /var/run/docker.sock:/var/run/docker.sock \
   gitlab/gitlab-runner:latest
```

 - Runner is need to be registered
```bash
gitlab-ci:~#  docker exec -it gitlab-runner gitlab-runner register
Running in system-mode.

Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://<your-vm-ip>
Please enter the gitlab-ci token for this runner:
fctLK91ibydMHdbJxRn9
Please enter the gitlab-ci description for this runner:
[18eeb3ad5ab8]: gitlab-ci-runner
Please enter the gitlab-ci tags for this runner (comma separated):
linux,xenial,ubuntu,docker
Whether to run untagged builds [true/false]:
[false]: true
Whether to lock the Runner to current project [true/false]:
[true]: false
Registering runner... succeeded                     runner=fctLK91i
Please enter the executor: docker-ssh, virtualbox, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, shell, ssh:
docker
Please enter the default Docker image (e.g. ruby:2.1):
alpine:latest
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

 - Runner is available and get ready: 
   - `Settings` -> `CI/CD` -> `Runners settings`, Runners activated for this project
   - `CI/CD` -> `Pipelines`, status = passed
   
 - Create new project `reddit` in group `homework`: `New project` -> `Create project`
```bash
Project path: http://<your-vm-ip>/reddit
Project name: reddit
Description: Monolith application from Artemmkin
Visibility Level: private
```

```bash
~$ git clone https://github.com/Artemmkin/reddit.git artemmkin-reddit
~$ cd artemmkin-reddit/
~$ git remote add gitlab http://<your-vm-ip>/homework/reddit.git
~$ git push -u gitlub monolith
```

 - Add file `.gitlab-ci.yml` that is used by GitLab Runner to manage project's jobs
```bash
image: ruby:2.4.2

stages:
  - test

variables:
  DATABASE_URL: 'mongodb://mongo/user_posts'

before_script:
  - bundle install

test:
  stage: test
  services:
    - mongo:latest
  script:
    - ruby simpletest.rb
```

 - Add file `simpletest.rb` with test functions that invoke from `.gitlab-ci.yml`
```bash
require_relative './app'
require 'test/unit'
require 'rack/test'

set :environment, :test

class MyAppTest < Test::Unit::TestCase
    include Rack::Test::Methods

    def app
       Sinatra::Application
    end
    def test_get_request
        get '/'
        assert last_response.ok?
    end
end
```

 - Include test library `rack-test` to the Gemfile
```bash
source 'https://rubygems.org'

gem 'sinatra'
gem 'haml'
gem 'bson_ext'
gem 'bcrypt'
gem 'puma'
gem 'mongo'
gem 'rack-test'
...
```

 - All changes should be pushed in your custom [GitLab CI](https://about.gitlab.com)
 - See that runner is not active for project `reddit`: `CI/CD` -> `Pipelines`, status = pending
 - Let's fix it and run runner: `Settings` -> `CI/CD` -> `Runners settings` -> `Enable for this project`
 - All next changes will use `Runner` and run test phase in pipeline
