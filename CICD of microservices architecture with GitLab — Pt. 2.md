# CI/CD of microservices architecture with GitLab — Pt. 2

## How we used GitLab‘s dynamic child pipelines to maintain and expand our services to many regions

> This part describes how we expand and improve our CI.  
> For part 1 — [https://medium.com/@aviadlevy/ci-cd-of-microservices-architecture-with-gitlab-bebce3735ccd](https://medium.com/@aviadlevy/ci-cd-of-microservices-architecture-with-gitlab-bebce3735ccd)

In the previous blog post I described how we used GitLab CI features to easily maintain many microservices life cycle — from packaging to delivering. We packaged the code in docker container, fully tested the docker image and deployed it to AWS EKS cluster.

As we continued growing and wanted to expand to new AWS regions, we’ve encountered a new challenge. In the next sections I’ll describe the challenge and how we solved it.

# The Challenge

At the time, on each microservice project we kept its own k8s’ deployment configurations per site, so if we wanted to add a new region we needed to add another deployment configuration per each new environment for each microservice project. When you’re dealing with ~80 microservices it can take a lot of time.

So the first thing we understood very fast was that we need to centralize our k8s configurations in one repository which will control everything. But if we want to initiate a new region we’ll have to create, for each microservice, its own new k8s configuration for the new region. Also in order to control which microservice to deploy on each region we’ll have to hold a very long `.gitlab-ci.yaml` with job for each microservice per each region, with specific `rules — changes:` which makes the maintenance for the CI very hard.

_We wanted to do better._

# How We Solved It

First thing we did was to “generalize” our k8s configuration. we want to maintain one k8s deployment configuration, and control the difference between the clusters programmatically (with simple search and replace for anchors)

Here is an example of a “generic” k8s deployment configuration (the anchors are `{{NAMESPACE}}` and `{{ECR_REGION}}`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app-xx
  name: app-xx
  namespace: {{NAMESPACE}}
spec:...
    spec:
      containers:
        - name: app-xx
          image: xxxxxxxx.dkr.ecr.{{ECR_REGION}}.amazonaws.com/group/app-xx:latest
          imagePullPolicy: IfNotPresent
          env:
...
      nodeSelector:
        node-role/nodes-{{NAMESPACE}}: "true"
      tolerations:
        - effect: NoSchedule
          key: dedicated
          operator: Equal
          value: nodes-{{NAMESPACE}}
        - effect: NoExecute
          key: dedicated
          operator: Equal
          value: nodes-{{NAMESPACE}}
```

This helps us to maintain only one deployment for each microservices, and all other regions that will be added in the future will also use this deployment configuration.

In order to use small maintainable `.gitlab-ci.yaml` we started using a feature introduced as a in GitLab on version 12.9 — “[Dynamic child pipelines](https://docs.gitlab.com/ee/ci/pipelines/parent_child_pipelines.html#dynamic-child-pipelines)”.

> Instead of running a child pipeline from a static YAML file, you can define a job that runs your own script to generate a YAML file, which is then used to trigger a child pipeline.

This feature allows us to create `.gitlab-ci.yaml` configuration from script and template and then run it as child pipeline.

This is how the `.gitlab-ci.yaml` looks:

```yaml
image: $CI_REGISTRY_IMAGE/$CI_IMAGE_NAME:latest
services:
  - docker:dind

stages:
  - generate-ci
  - triggers

generate-ci:
  stage: generate-ci
  script:
    - python3 scripts/generate_ci.py
  artifacts:
    paths:
      - ci-out
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
      when: never
    - when: always

parents:
  stage: triggers
  trigger:
    include:
      - artifact: ci-out/middle-ci.yml
        job: generate-ci
    strategy: depend
  variables:
    PARENT_PIPELINE_ID: $CI_PIPELINE_ID
    PARENT_JOB_NAME: generate-ci
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
      when: never
    - when: always
```

On `generate-ci` stage we’re using a simple python script. The script gets only the relevant git changes and dynamically builds the `middle-ci.yaml` file. Then, the `parents` stage will use this file and will run the necessary stages to update deployment configuration with the matching GitLab’s environment variables.

This is the [Jinja](https://pypi.org/project/Jinja2/) template we’re using:

```yaml

image: gitlab.locsec.net:4567/cloudmta/threatcloud-mta-infra/ci-image:latest
# https://docs.gitlab.com/ee/ci/yaml/index.html#switch-between-branch-pipelines-and-merge-request-pipelines
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS && $CI_PIPELINE_SOURCE == "push"
      when: never
    - when: always

variables:
  PY_COLORS: '1'
  GIT_SUBMODULE_STRATEGY: recursive

stages:
{% for environment in environments.keys() %}
  - "validate:{{ environment }}:{{ stage_params[1:] |join('-') }}"
  - "deploy:{{ environment }}:{{ stage_params[1:] |join('-') }}"
{% endfor %}

{% for stage in ["validate", "deploy"] %}
{% for environment, regions in environments.items() %}
{% for region in regions %}
{{ stage }}:{{ resource }}:{{ environment }}:{{ region }}:{{ stage_params[1:] |join('-') }}:
{% if stage == "deploy" %}
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH
      when: never
{% if ci_project_name == "app-infra" %}
    - when: manual
{% else %}
    - when: on_success
{% endif %}
  environment:
    name: {{ stage_params[1:] |join('-') }}/{{ resource }}/{{ environment }}/{{ region }}
{% endif %}
  variables:
    AWS_ACCESS_KEY_ID: ${{ region | upper | replace("-", "_") }}_{{ environment | upper }}_STS_KEY
    AWS_SECRET_ACCESS_KEY: ${{ region | upper | replace("-", "_") }}_{{ environment | upper }}_STS_SECRET
    CLUSTER_NAME: {{ region }}-{{ "prd" if environment == "prod" else environment }}
    K8S_NAMESPACE: group-{{ "dev" if environment == "stg" else environment }}
    AWS_REGION: {{ region }}
  stage: {{ stage }}:{{ environment }}:{{ stage_params[1:] |join('-') }}
  allow_failure: false
  script:
{% if ci_project_name != "app-infra" %}
    - git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.locsec.net/group/app-infra.git
    - cd app-infra
{% endif %}
    - aws --region $AWS_REGION eks update-kubeconfig --name $CLUSTER_NAME
    - kubectl config set-context --current --namespace=$K8S_NAMESPACE
    - python3 scripts/apply_k8s.py --file {{ file }} --resource {{ resource }} --environment {{ environment }} --region {{ region }} --stage {{ stage }}
{% endfor %}
{% endfor %}
{% endfor %}
```

You can see in the example where the “magic” is hidden. In the `variables` section we have our AWS environment variable, which is needed to deploy our app to the EKS cluster. This environment is built from the template `${{ region | upper | replace(“-”, “_”) }}_{{ environment | upper }}_STS_KEY`.

Here is an output example from this template:

```yaml

image: gitlab.locsec.net:4567/cloudmta/threatcloud-mta-infra/ci-image:latest
# https://docs.gitlab.com/ee/ci/yaml/index.html#switch-between-branch-pipelines-and-merge-request-pipelines
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS && $CI_PIPELINE_SOURCE == "push"
      when: never
    - when: always
variables:
  PY_COLORS: '1'
  GIT_SUBMODULE_STRATEGY: recursive

stages:
  - "validate:stg:app-config-service"
  - "deploy:stg:app-config-service"
  - "validate:prod:app-config-service"
  - "deploy:prod:app-config-service"



validate:deployment:stg:eu-west-1:app-config-service:
  variables:
    AWS_ACCESS_KEY_ID: $EU_WEST_1_STG_STS_KEY
    AWS_SECRET_ACCESS_KEY: $EU_WEST_1_STG_STS_SECRET
    CLUSTER_NAME: eu-west-1-group-stg
    K8S_NAMESPACE: group-stg
    AWS_REGION: eu-west-1
  stage: validate:stg:app-config-service
  allow_failure: false
  script:
    - aws --region $AWS_REGION eks update-kubeconfig --name $CLUSTER_NAME
    - kubectl config set-context --current --namespace=$K8S_NAMESPACE
    - python3 scripts/apply_k8s.py --file k8s/app-config-service/deployment.yaml --resource deployment --environment stg --region eu-west-1 --stage validate

validate:deployment:prod:us-east-1:app-config-service:
  variables:
    AWS_ACCESS_KEY_ID: $US_EAST_1_PROD_STS_KEY
    AWS_SECRET_ACCESS_KEY: $US_EAST_1_PROD_STS_SECRET
    CLUSTER_NAME: us-east-1-group-prd
    K8S_NAMESPACE: group-prod
    AWS_REGION: us-east-1
  stage: validate:prod:app-config-service
  allow_failure: false
  script:
    - aws --region $AWS_REGION eks update-kubeconfig --name $CLUSTER_NAME
    - kubectl config set-context --current --namespace=$K8S_NAMESPACE
    - python3 scripts/apply_k8s.py --file k8s/app-config-service/deployment.yaml --resource deployment --environment prod --region us-east-1 --stage validate


deploy:deployment:stg:eu-west-1:app-config-service:
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH
      when: never
    - when: manual
  environment:
    name: app-config-service/deployment/stg/eu-west-1
  variables:
    AWS_ACCESS_KEY_ID: $EU_WEST_1_STG_STS_KEY
    AWS_SECRET_ACCESS_KEY: $EU_WEST_1_STG_STS_SECRET
    CLUSTER_NAME: eu-west-1-group-stg
    K8S_NAMESPACE: group-dev
    AWS_REGION: eu-west-1
  stage: deploy:stg:app-config-service
  allow_failure: false
  script:
    - aws --region $AWS_REGION eks update-kubeconfig --name $CLUSTER_NAME
    - kubectl config set-context --current --namespace=$K8S_NAMESPACE
    - python3 scripts/apply_k8s.py --file k8s/app-config-service/deployment.yaml --resource deployment --environment stg --region eu-west-1 --stage deploy

deploy:deployment:prod:us-east-1:app-config-service:
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH
      when: never
    - when: manual
  environment:
    name: app-config-service/deployment/prod/us-east-1
  variables:
    AWS_ACCESS_KEY_ID: $US_EAST_1_PROD_STS_KEY
    AWS_SECRET_ACCESS_KEY: $US_EAST_1_PROD_STS_SECRET
    CLUSTER_NAME: us-east-1-group-prd
    K8S_NAMESPACE: group-prod
    AWS_REGION: us-east-1
  stage: deploy:prod:app-config-service
  allow_failure: false
  script:
    - aws --region $AWS_REGION eks update-kubeconfig --name $CLUSTER_NAME
    - kubectl config set-context --current --namespace=$K8S_NAMESPACE
    - python3 scripts/apply_k8s.py --file k8s/app-config-service/deployment.yaml --resource deployment --environment prod --region us-east-1 --stage deploy
```

Adding a new region, to deploy all our microservices, is as easy as adding a Gitlab’s group environment variable!

This is how the pipeline will look on the centralize k8s configuration project:

![](https://miro.medium.com/max/875/1*lG8gvALfRfZRNmPPlcZuuw.png)

dynamic child pipelines

From each project, we’ll use this stages to deploy only the specific microservice:

```yaml

create deploy to dedicated prod:
  stage: pre-prod-deploy
  image: gitlab.locsec.net:4567/group/app-infra/ci-image:latest
  script:
    - git clone https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.locsec.net/group/app-infra.git
    - cd app-infra
    - python3 scripts/generate_ci.py
    - cp -r ci-out ..
  artifacts:
    paths:
      - ci-out
  except:
    refs:
      - schedules
  only:
    refs:
      - master


deploy to dedicated prod:
  stage: prod-deploy
  variables:
    PARENT_PIPELINE_ID: $CI_PIPELINE_ID
    PARENT_JOB_NAME: "create deploy to dedicated prod"
  trigger:
    include:
      - artifact: ci-out/middle-ci.yml
        job: "create deploy to dedicated prod"
    strategy: depend
  when: on_success
  except:
    refs:
      - schedules
  only:
    refs:
      - master
```

That’s it. we can now easily maintain our k8s configurations, our deployment CI and initiate new regions.

# Conclusion

In this post I continued describing how GitLab CI helps us with our app growth. I tried to keep it as simple as possible, but if something is not clear enough, feel free to contact me with any question you have.

I hope you found this tutorial helpful. Thanks for reading!

> Thanks a lot to [Israelst](https://medium.com/@israelst11) for reviewing this post
