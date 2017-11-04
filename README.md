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

 - Open [http://\<your-vm-ip\>](http://\<your-vm-ip\>) and create new administrative `root` account
 - Authorized like `root` and login in GitLab CI
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

## Homework 20

 - Get ready to deploy [GitLab CI](https://about.gitlab.com) environment from Docker in hw 19
 - File `.gitlab-ci.yml` defines pipeline stages: `Build`, `Test` and `Review`. 
 Let's edit `Review` stage in `.gitlab-ci.yml` and define `dev` environment
```bash
stages:
  - build
  - test
  - review

build:
  stage: build
  script:
    - echo 'Building'

test unit:
  stage: test
  script:
    - echo 'Unit Testing 1'

test integration:
  stage: test
  script:
    - echo 'Integration Testing 2'

dev:
  stage: review
  script:
    - echo 'Deploy to Dev'
  environment:
    name: dev
    url: http://dev.example.com # define GitLab CI to use external URL for this job
```

 - You can look here `CI/CD` -> `Environments` = `dev`
 - Add new pipeline stages: `stage` and `production`: 
   - The new stages do sensitive actions and let's to use manual control
   - Use tagging with number version in name to add stages `stage` and `production` in pipeline
 ```bash
stages:
  - build
  - test
  - review
  - stage
  - production  
...

staging:
  stage: stage
  when: manual
  only:
    - /^\d+\.\d+.\d+/
  script:
    - echo 'Deploy'
  environment:
    name: stage
    url: https://beta.example.com

production:
  stage: production
  when: manual
  only:
    - /^\d+\.\d+.\d+/
  script:
    - echo 'Deploy'
  environment:
    name: production
    url: https://example.com

 ```

  - Push project changes to the remote Git repository and look at pipeline stages: 
  `build`, `test`, `review`; it is not a full pipeline
  - Create new tag, push changes to the remote Git repository and look at pipeline stages again: 
  `build`, `test`, `review`, `stage` and `production`; right now it is a full pipeline
```bash
$ git tag 2.4.10
$ git push --tags
Total 0 (delta 0), reused 0 (delta 0)
To http://104.199.11.69/homework/gitlab-ci-01-homework.git
 * [new tag]         2.4.10 -> 2.4.10
```

  - [GitLab CI](https://about.gitlab.com) might have a dynamic environment for each feature/branch. 
  It is very useful to get a separate environment for testing from main branch. 
  Let's define new job with variables for each branch except `master`.
```bash
...
branch review:
  stage: review
  script:
    - echo "Deploy to $CI_ENVIRONMENT_SLUG"
  environment:
    name: branch/$CI_COMMIT_REF_NAME
    url: http://$CI_ENVIRONMENT_SLUG.example.com
  only:
    - branches
  except:
    - master
...
```

 - Create two new branches, for example `feature/issue-001` and `feature/issue-002` 
 and push them to the remote Git repository
 ```bash
$ git branch
  feature/issue-001
  feature/issue-002
* master
 ``` 

 -  `CI/CD` -> `Environments`, open directory `branch` to look at our branches
 
 - Example file
```bash
stages:
  - build
  - test
  - review
  - stage
  - production

build:
  stage: build
  script:
    - echo 'Building'

test unit:
  stage: test
  script:
    - echo 'Unit Testing 1'

test integration:
  stage: test
  script:
    - echo 'Integration Testing 2'

dev:
  stage: review
  script:
    - echo 'Deploy to Dev'
  environment:
    name: dev
    url: http://dev.example.com # define GitLab CI to use external URL for this job

branch review:
  stage: review
  script:
    - echo "Deploy to $CI_ENVIRONMENT_SLUG"
  environment:
    name: branch/$CI_COMMIT_REF_NAME
    url: http://$CI_ENVIRONMENT_SLUG.example.com
  only:
    - branches
  except:
    - master

stage:
  stage: stage
  when: manual # run job only manual from ui of GitLab CI
  only:
    - /^\d+\.\d+.\d+/ # filter the commits match the pattern of tags (example: tag: 2.4.10)
  script:
    - echo 'Deploy to Stage'
  environment:
    name: stage
    url: https://beta.example.com # define GitLab CI to use external URL for this job

production:
  stage: production
  when: manual # run job only manual from ui of GitLab CI
  only:
    - /^\d+\.\d+.\d+/ # filter the commits match the pattern of tags (example: tag: 2.4.10)
  script:
    - echo 'Deploy to Production'
  environment:
    name: production
    url: https://example.com # define GitLab CI to use external URL for this job

```

=======  
If you want to reset user password, enable authentication config or restart all components of [GitLab CI](https://about.gitlab.com)
 use console `gitlab-rails`:
```bash
home: ~$ docker-machine ssh gitlab-ci

docker-user@gitlab-ci:~$ sudo -i
root@gitlab-ci:~# docker ps
root@gitlab-ci:~# docker exec -it <gitlab-ce_docker_id> /bin/bash
root@gitlab:/# gitlab-rails console

irb(main):001:0> ApplicationSetting.last.update_attributes(password_authentication_enabled: true)

irb(main):002:0> user = User.where(id: 1).first
irb(main):003:0> user.password = 'secret_pass'
irb(main):004:0> user.password_confirmation = 'secret_pass'
irb(main):005:0> user.save!

irb(main):006:0> quit

root@gitlab:/# gitlab-ctl restart
```

