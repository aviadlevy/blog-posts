CI/CD of microservices architecture with GitLab
===============================================

How we made our microservices continuous integration & delivery process easily maintained
-----------------------------------------------------------------------------------------

> This is a re-publish for a post I wrote back in Aug 2019. many things changed since then and I hope to write new post about it soon

> To see how we further improved our CI/CD process with GitLab, check out [part 2](https://medium.com/@aviadlevy/ci-cd-of-microservices-architecture-with-gitlab-pt-2-f640d4c57c9b)

As the Threat Cloud Application team, we’re responsible for many cloud applications. We have public applications, and many other internal applications that support them. This software design is also known as a microservices architecture. Our applications are deployed on Kubernetes cluster which is deployed in AWS.

We started using GitLab as our central source code management and application life cycle about a year ago.

In this post I’ll describe our evolution of understanding how to work with GitLab in order to easily maintain the life cycle of our microservices.

What we had when we began
=========================

When we joined our internal hosted GitLab our main focus was to simply import all of our code base from a simple git server to GitLab. We didn’t fully invest in exploring the many ways GitLab can help us in the development cycle and the CI/CD process.

It wasn’t long before we began designing our CI/CD process, but even then we didn’t fully capitalize the GitLab features and our .gitlab-ci.yml looked like this:

```yaml
image: docker:latest
services:
- docker:dind

stages:
- build
- integration-test
- push
- dev-deploy
- prod-deploy

# Cache downloaded dependencies and plugins between builds.
cache:
  paths:
  - .m2/repository
  key: "$CI_JOB_NAME"

variables:
  CONTAINER_RELEASE_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  CONTAINER_RELEASE_IMAGE_LATEST: $CI_REGISTRY_IMAGE:latest
  DEV_KUBECTL_CONFIG: $k8s_deploy_config_kube1
  EU_PROD_KUBECTL_CONFIG: $k8s_deploy_config_eu_west_1_kube2
  US_PROD_KUBECTL_CONFIG: $k8s_deploy_config_us_east_1_kube1
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"


maven-build:
  image: snirnx/docker-maven:3.5.4
  stage: build
  script:
  - "sh ci_scripts/run_build.sh dev jar"
  except:
    - master

maven-build-master:
  image: snirnx/docker-maven:3.5.4
  stage: build
  script:
  - "sh ci_scripts/run_build.sh master jar"
  only:
    - master

integration:
  image: snirnx/docker-maven:3.5.4
  stage: integration-test
  script:
    - "bash ci_files/integration_test.sh $CI_COMMIT_SHA"
  artifacts:
    name: "CI_COMMIT_SHA-results"
    when: always
    expire_in: 4 days
    paths:
      - reputation-service-integration-tests/target/site/
      - reputation-service-integration-tests/log/
  tags:
    - reputation-integration-tests


development-deploy:
  stage: dev-deploy
  image: snirnx/kubectl:v1.8.6
  script:
  - "sh ci_scripts/deploy.sh dev eu"
  only:
  - master

production-deploy:
  stage: prod-deploy
  image: snirnx/kubectl:v1.8.6
  script:
  - "sh ci_scripts/deploy.sh prod us"
  - "sh ci_scripts/deploy.sh prod eu"  
  only:
  - master
  when: manual
```

As we can see, the pipeline was depend fully on old-style bash scripts.

```bash
ENV=$1
APP=$2
PACKAGE=$3

mvn clean install
rc=$?
if [[ $rc -ne 0 ]]; then
  echo "Maven build failed"; exit $rc
fi

cp target/$APP.$PACKAGE docker/

docker build -t $APP docker/

docker images

#echo "docker login"
docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

echo "docker tag"
docker tag $APP $CI_REGISTRY_IMAGE/$APP:$CI_COMMIT_SHA
if [[ "$ENV" != "dev" ]]; then
    docker tag $APP $CI_REGISTRY_IMAGE/$APP:latest
fi

#echo "docker push"
docker push $CI_REGISTRY_IMAGE/$APP:$CI_COMMIT_SHA
if [[ "$ENV" != "dev" ]]; then
    docker push $CI_REGISTRY_IMAGE/$APP:latest
fi

rc=$?
if [[ $rc -ne 0 ]]; then
  echo "Failed to push to regsitry $APP"; exit $rc
fi


echo "Pushed build '$CI_REGISTRY_IMAGE/$APP:$CI_COMMIT_SHA'"
```

One big downside of using this kind of scripts inside .gitlab-ci.yml job was that we needed to handle exit code of each ourselves, or else the job won’t fail when one step is failed. Other downside was that GitLab didn’t highlight each step from within a single script, so we needed to handle it ourselves using echo commands.

The only thing we considered as an advantage was the ability to copy the generic bash script between the different projects, allowing our developers to focus on the code of the new microservice application and not on the CI process. Or so we thought. But what happen when you want to improve something in your CI bash scripts? Add a new feature? Going over ~30 repositories is not an easy thing.

One solution was to keep one scripts project in GitLab with all the bash scripts allowing each project to clone and use the script it requires. Even though this is good enough solution, we knew there had to be more native GitLab way to handle centralize CI/CD pipeline.

What we have now
================

We start using a feature introduced as a Premium feature on the GitLab 10.5 and moved to GitLab core on 11.4 release. The feature is called include.

> Using the include keyword, you can allow the inclusion of external YAML file

With this feature our .gitlab-ci.yml looks like this:

```yaml
image: docker:latest
services:
  - docker:dind

stages:
  - build
  - integration-test
  - pre-dev-deploy
  - dev-deploy
  - continue-to-prod
  - pre-prod-deploy
  - prod-deploy
  - post-deploy

# Cache downloaded dependencies and plugins between builds.
cache:
  paths:
    - .m2/repository
  key: "$CI_JOB_NAME"

variables:
  IMAGE_NAME: "image-name"
  DOCKER_FILE_DIR: docker/
  K8S_DEV_CONFIGMAP: ""
  K8S_PROD_CONFIGMAP: ""
  K8S_DEV_DEPLOYMENT: "k8s/dev/deployment.yaml"
  K8S_PROD_DEPLOYMENT: "k8s/prod/deployment.yaml"

include:
  - project: "ThreatCloud/centralize-reputation-pipeline"
    file: .gitlab-ci-build-mvn.yml
  - project: "ThreatCloud/centralize-reputation-pipeline"
    file: .gitlab-ci-integration.yml
  - project: "ThreatCloud/centralize-reputation-pipeline"
    file: .gitlab-ci-deploy.yml
  - project: "ThreatCloud/centralize-reputation-pipeline"
    file: .gitlab-ci-post.yml

```

The first thing we did was to define what stages we wanted to be included in our pipeline for all our applications:

*   **Build** — Run unit tests, dockerize the service and also test that the yamls of the deployments are valid
*   **Integration Tests** — Run the integration test for each service
*   **Pre Staging deployment** — Things needed to be deployed in the staging k8s before deploying the actual service, e.g. configmap
*   **Staging deployment** — Deploying the service to staging k8s
*   **Pre Production deployment** — Things needed to be deployed in the production k8s before deploying the actual service, e.g. configmap
*   **Production deployment** — Deploying the service to production k8s
*   **Post Production deployment** — Things we’re doing after each production deployment, e.g. notify to slack about the deployment and release changes

Then we built .gitlab-ci.yml files for all steps. We made sure to keep it generic so that only by using different variables defined in each project the pipeline would work as expected.

Now we can enjoy a native GitLab CI execution and highlight in all of our jobs. We can easily add new features to each step and automatically all the projects will start enjoying it automatically. The developer doesn’t need to deal with the CI/CD process and can focus solely on writing new code. All he needs to do is to copy the .gitlab-ci.yml, change a few variable and that’s it, the pipeline works.

Conclusion
==========

In this post I tried, as simple as possible, to describe the process we went through to successfully maintain our CI/CD pipelines with GitLab. I’m sure that there are other solutions and ways that GitLab can help improve an application life cycle.

I hope that you found this tutorial helpful. Thanks for reading!
