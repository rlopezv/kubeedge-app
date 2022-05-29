# kubeedge-app
Rerpository for kubeedge applications.

This repository is only for evidence. It contains the required content to implement a CI workflow for an application to be deployed in Kubeedge.


It contains two projects:

- [nginx-app](nginx-app), a simple nginx application.
- [nginx-infra](nginx-infra), descriptors required for provisioning the application using kubeedge.

Since gitlab is used for implementing the CI workflow each project will be included in the gitlab instance in their own repository, like shown in the following image:

![gitops-repos](img/gitops-repositories.png)


## nginx-app

Simple nginx-app that shows information about the execution context. Can be executed without root privileges (port>1024)

How to build
```
$ docker build . -t nginx-app
```

How to run:
```
$ docker run -P -d nginx-app
```

http://localhost:8080/demo


![nginx-app](img/nginx-app.png)

A .gitlab-ci.yml has been included to define the required pipeline for compiling, publishing image and update infrastructure repository as shown in image.

![ci-pipeline](img/ci-pipeline.png)

```yaml
image: docker:stable-git

variables:
  CI_REGISTRY_IMAGE: "rlopezv/kubeedge-demo-app"
  CI_INFRA_REPO: "https://${GITLAB_USER_LOGIN}:${CI_ACCESS_TOKEN}@gitlab.com/ramon.lopez.viana/kubeedge-demo-infra.git"
  CI_INFRA_PATH: "kubeedge-demo-infra"

## Defined stages
stages:
  - build
  - test
  - publish

environment:
  stage: .pre 
  script:
    - printenv

build:
  image: docker:stable-dind
  stage: build
  before_script:
    - docker info
  script:
    - docker build -t ${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG} .
  rules:
    - when: never

build_kaniko:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    # - echo "{\"auths\":{\"${CI_REGISTRY}\":{\"auth\":\"$(printf "%s:%s" "${CI_REGISTRY_USER}" "${CI_REGISTRY_PASSWORD}" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json
    - echo ${CI_DOCKER_AUTH_CONFIG} > /kaniko/.docker/config.json 
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}"
  rules:
    - when: always

test:
  stage: test
  script:
    - echo 'tests should be here'

update_infra:
  stage: publish
  before_script:
    - git config --global user.email "${GITLAB_USER_EMAIL}"
    - git config --global user.name "${GITLAB_USER_NAME}"
  script:
    - git clone $CI_INFRA_REPO
    - cd $CI_INFRA_PATH
    - echo ${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}@${CI_COMMIT_BEFORE_SHA} >> versions.txt
    - cat versions.txt
    - git add -A
    - git commit -m "Pushed from ${CI_PROJECT_URL} ${CI_COMMIT_REF_SLUG} ${CI_COMMIT_BEFORE_SHA}" 
    - git push $CI_INFRA_REPO
```

Since the application will be deployed in rpi (arm64) a runner has been created and registered in gitlab for such purpose.

![Runner](img/gitlab-rummer-register.png)


Two stages requires attention build and publish. 

#### Build

In this stage is implemented the building and publishing of the image (using kaniko)


![build](img/ci-pipline-publish-image.png)


#### Publish

In this stage is implemented the update of the infrastructure repo that will trigger the CD process.


![build](img/ci-pipeline-update-infra.png)


## nginx-infra

This repository contains the descriptors required for the application deployment in the kubeedge environment.





