# rancher-mysql-demo

## Overview

The following diagram shows the relationship of the Rancher apps, docker containers, and code in this Rancher demonstration.

![Image of architecture](architecture.png)

### Contents

1. [Expectations](#expectations)
    1. [Space](#space)
    1. [Time](#time)
    1. [Background knowledge](#background-knowledge)
1. [Demonstrate](#demonstrate)
    1. [Clone repository](#clone-repository)
    1. [Prerequisites](#prerequisites)
    1. [Set environment variables](#set-environment-variables)
    1. [Create custom answer files](#create-custom-answer-files)
    1. [Create custom kubernetes configuration files](#create-custom-kubernetes-configuration-files)
    1. [Set default context](#set-default-context)
    1. [Add catalogs](#add-catalogs)
    1. [Create project](#create-project)
    1. [Switch context](#switch-context)
    1. [Create namespace](#create-namespace)
    1. [Create persistent volume](#create-persistent-volume)
    1. [Install Kafka](#install-kafka)
    1. [Install Kafka test client](#install-kafka-test-client)
    1. [Install mySQL](#install-mysql)
    1. [Install phpMyAdmin](#install-phpmyadmin)
    1. [Install mock-data-generator](#install-mock-data-generator)
    1. [Install stream-loader](#install-stream-loader)
    1. [Install senzing-api-server](#install-senzing-api-server)
    1. [Test Senzing REST API server](#test-senzing-rest-api-server)
1. [Cleanup](#cleanup)
    1. [Switch context for delete](#switch-context-for-delete)
    1. [Delete everything in project](#delete-everything-in-project)
    1. [Default context after cleanup](#default-context-after-cleanup)
    1. [Delete catalogs](#delete-catalogs)

## Expectations

### Space

This repository and demonstration require 20 GB free disk space.

### Time

Budget 4 hours to get the demonstration up-and-running, depending on CPU and network speeds.

### Background knowledge

This repository assumes a working knowledge of:

1. [Docker](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/docker.md)
1. [Kubernetes](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/kubernetes.md)
1. [Helm](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/helm.md)
1. [Rancher](https://github.com/Senzing/knowledge-base/blob/master/WHATIS/rancher.md)

## Demonstrate

### Clone repository

1. Using these environment variable values:

    ```console
    export GIT_ACCOUNT=senzing
    export GIT_REPOSITORY=rancher-demo
    ```

   Then follow steps in [clone-repository](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/clone-repository.md).

1. After the repository has been cloned, be sure the following are set:

    ```console
    export GIT_ACCOUNT_DIR=~/${GIT_ACCOUNT}.git
    export GIT_REPOSITORY_DIR="${GIT_ACCOUNT_DIR}/${GIT_REPOSITORY}"
    ```

### Prerequisites

#### Rancher

1. [Install Rancher](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-rancher.md).

1. Simple example for local development:

    ```console
    sudo docker run \
      --volume /opt/rancher:/var/lib/rancher \
      --publish 80:80 \
      --publish 443:443 \
       rancher/rancher:latest
    ```

#### Rancher cluster

1. [Create the cluster](https://rancher.com/docs/rancher/v2.x/en/quick-start-guide/deployment/quickstart-manual-setup/#4-create-the-cluster)
1. Set environment variable with Rancher cluster name. Example:

    ```console
    export RANCHER_CLUSTER_NAME=my-rancher-cluster
    ```

#### Senzing docker images

1. Make Senzing docker images.

    ```console
    export BASE_IMAGE=senzing/python-mysql-base

    sudo docker build \
      --tag ${BASE_IMAGE} \
      https://github.com/senzing/docker-python-mysql-base.git

    sudo docker build \
      --tag senzing/stream-loader \
      --build-arg BASE_IMAGE=${BASE_IMAGE} \
      https://github.com/senzing/stream-loader.git

    sudo docker build \
      --tag senzing/mock-data-generator \
      https://github.com/senzing/mock-data-generator.git

    sudo docker build \
      --tag senzing/mysql-init \
      https://github.com/senzing/docker-mysql-init.git
    ```

1. Build [senzing/senzing-api-server](https://github.com/Senzing/senzing-api-server#using-docker) docker image.

#### Docker registry

1. If you need to create a private docker registry, see
       [HOWTO - Install docker registry server](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/install-docker-registry-server.md).
1. Set environment variable. Example:

    ```console
    export DOCKER_REGISTRY_URL=my.docker-registry.com:5000
    ```

1. Add Senzing docker images to private docker registry.

    ```console
    for GIT_REPOSITORY in \
      "mock-data-generator" \
      "mysql-init" \
      "senzing-api-server" \
      "stream-loader"; \
    do \
      sudo docker tag senzing/${GIT_REPOSITORY} ${DOCKER_REGISTRY_URL}/senzing/${GIT_REPOSITORY}; \
      sudo docker push ${DOCKER_REGISTRY_URL}/senzing/${GIT_REPOSITORY}; \
      sudo docker rmi  ${DOCKER_REGISTRY_URL}/senzing/${GIT_REPOSITORY}; \
    done
    ```

### Set environment variables

1. Environment variables that need customization.  Example:

    ```console
    export RANCHER_CLUSTER_NAME=my-rancher-cluster
    export RANCHER_PREFIX=my-senzing-mysql
    ```

1. Set environment variables listed in "[Clone repository](#clone-repository)".
1. Environment variables used in `rancher` CLI commands.

    ```console
    export RANCHER_PROJECT_NAME=${RANCHER_PREFIX}-project
    export RANCHER_NAMESPACE_NAME=${RANCHER_PREFIX}-namespace
    ```

### Create custom answer files

1. Variation #1. Quick method using `envsubst`.

    ```console
    export RANCHER_ANSWERS_DIR=${GIT_REPOSITORY_DIR}/rancher-answers
    mkdir -p ${RANCHER_ANSWERS_DIR}

    for file in ${GIT_REPOSITORY_DIR}/rancher-answers-templates/*.yaml; \
    do \
      envsubst < "${file}" > "${RANCHER_ANSWERS_DIR}/$(basename ${file})";
    done
    ```

1. Variation #2. Manually copy example files and modify. Example:

    ```console
    export RANCHER_ANSWERS_DIR=${GIT_REPOSITORY_DIR}/rancher-answers
    mkdir -p ${RANCHER_ANSWERS_DIR}
    cp ${GIT_REPOSITORY_DIR}/rancher-answers-templates/*.yaml ${RANCHER_ANSWERS_DIR}
    ```

    1. Modify ${RANCHER_ANSWERS_DIR}/mock-data-generator.yaml
        1. **image.repository**
            1. Example: `'image.repository': "my.docker-registry.com:5000/senzing/mock-data-generator"`
        1. **senzing.kafkaBootstrapServerHost**
            1. Example: `'senzing.kafkaBootstrapServerHost': "my-senzing-mysql-kafka-kafka"`
    1. Modify ${RANCHER_ANSWERS_DIR}/phpmyadmin.yaml
        1. **db.host**
            1. Example: `'db.host': "my-senzing-mysql-mysql-mysql"`
    1. Modify ${RANCHER_ANSWERS_DIR}/senzing-api-server.yaml
        1. **image.repository**
            1. Example: `'image.repository': "my.docker-registry.com:5000/senzing/senzing-api-server"`
    1. Modify ${RANCHER_ANSWERS_DIR}/stream-loader-mysql.yaml
        1. **image.repository**
            1. Example: `'image.repository': "my.docker-registry.com:5000/senzing/stream-loader"`
        1. **senzing.kafkaBootstrapServerHost**
            1. Example: `'senzing.kafkaBootstrapServerHost': "my-senzing-mysql-kafka-kafka"`

### Create custom kubernetes configuration files

1. Variation #1. Quick method using `envsubst`.

    ```console
    export KUBERNETES_DIR=${GIT_REPOSITORY_DIR}/kubernetes
    mkdir -p ${KUBERNETES_DIR}

    for file in ${GIT_REPOSITORY_DIR}/kubernetes-templates/*; \
    do \
      envsubst < "${file}" > "${KUBERNETES_DIR}/$(basename ${file})";
    done
    ```

1. Variation #2. Manually copy example files and modify. Example:

    ```console
    export KUBERNETES_DIR=${GIT_REPOSITORY_DIR}/kubernetes-2
    mkdir -p ${KUBERNETES_DIR}
    cp ${GIT_REPOSITORY_DIR}/kubernetes-templates/*.yaml ${KUBERNETES_DIR}
    ```

    1. Modify ${KUBERNETES_DIR}/persistent-volume-claim-opt-senzing.yaml
        1. **namespace**
            1. Example: `namespace: my-senzing-mysql-namespace`

### Set default context

1. Switch context.  Example:

    ```console
    rancher context switch \
      Default
    ```

### Add catalogs

1. Add "Helm Stable"
    1. In Rancher > Global > Catalogs:
        1. Example URL is [https://localhost/g/catalog](https://localhost/g/catalog)
        1. "Enable" Helm Stable

1. Add Senzing catalog.  Example:

    ```console
    rancher catalog add \
      senzing \
      https://github.com/senzing/charts
    ```

### Create project

1. Example:

    ```console
    rancher projects create \
      --cluster ${RANCHER_CLUSTER_NAME} \
      --description "Project for ${RANCHER_PROJECT_NAME}" \
      ${RANCHER_PROJECT_NAME}
    ```

### Switch context

1. Example:

    ```console
    rancher context switch \
      ${RANCHER_PROJECT_NAME}
    ```

### Create namespace

1. Example:

    ```console
    rancher namespace create \
      --description "Namespace for ${RANCHER_PROJECT_NAME}" \
      ${RANCHER_NAMESPACE_NAME}
    ```

### Create persistent volume

1. If you do not already have an `/opt/senzing` directory on your system, visit
   [HOWTO - Create SENZING_DIR](https://github.com/Senzing/knowledge-base/blob/master/HOWTO/create-senzing-dir.md).

1. Create persistent volumes. Example:

    ```console
    rancher kubectl create \
      -f ${KUBERNETES_DIR}/persistent-volume-opt-senzing.yaml
    ```

1. Create persistent volume claims. Example:

    ```console
    rancher kubectl create \
      -f ${KUBERNETES_DIR}/persistent-volume-claim-opt-senzing.yaml
    ```

### Install Kafka

1. Example:

    ```console
    rancher app install \
      --answers ${RANCHER_ANSWERS_DIR}/kafka.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      library-kafka \
      ${RANCHER_PREFIX}-kafka
    ```

### Install Kafka test client

1. Install Kafka test client app. Example:

    ```console
    rancher app install \
      --answers ${RANCHER_ANSWERS_DIR}/kafka-test-client.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-kafka-test-client \
      ${RANCHER_PREFIX}-kafka-test-client
    ```

1. Run the test client. Run in a separate terminal window. Example:

    ```console
    export RANCHER_PREFIX=my-senzing-mysql
    export RANCHER_NAMESPACE_NAME=${RANCHER_PREFIX}-namespace

    rancher kubectl exec \
      -it \
      -n ${RANCHER_NAMESPACE_NAME} \
      ${RANCHER_PREFIX}-kafka-test-client -- /usr/bin/kafka-console-consumer \
        --bootstrap-server ${RANCHER_PREFIX}-kafka-kafka:9092 \
        --topic senzing-kafka-topic \
        --from-beginning
    ```

### Install mySQL

1. Example:

    ```console
    rancher app install \
      --answers ${RANCHER_ANSWERS_DIR}/mysql.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      library-mysql \
      ${RANCHER_PREFIX}-mysql
    ```

### Initialize database

1. Example:

    ```console
    rancher app install \
      --answers ${RANCHER_ANSWERS_DIR}/mysql-client.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-mysql-client \
      ${RANCHER_PREFIX}-mysql-client
    ```

### Install phpMyAdmin

1. Install phpMyAdmin app. Example:

    ```console
    rancher app install \
      --answers ${RANCHER_ANSWERS_DIR}/phpmyadmin.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-phpmyadmin \
      ${RANCHER_PREFIX}-phpmyadmin
    ```

1. Port forward to local machine.  Run in a separate terminal window. Example:

    ```console
    export RANCHER_PREFIX=my-senzing-mysql
    export RANCHER_NAMESPACE_NAME=${RANCHER_PREFIX}-namespace

    rancher kubectl port-forward \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      svc/${RANCHER_PREFIX}-phpmyadmin 8081:80
    ```

1. Open browser to [localhost:8081](http://localhost:8081)
    1. Login
       1. mysqlUser/mysqlPassword in `rancher-answers/mysql.yaml`
       1. Default: username: g2  password: g2
    1. On left-hand navigation, select "G2" database to explore.

### Install mock-data-generator

1. Example:

    ```console
    rancher app install \
      --answers ${RANCHER_ANSWERS_DIR}/mock-data-generator.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-senzing-mock-data-generator \
      ${RANCHER_PREFIX}-senzing-mock-data-generator
    ```

### Install stream-loader

1. Example:

    ```console
    rancher app install \
      --answers ${RANCHER_ANSWERS_DIR}/stream-loader-mysql.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-senzing-stream-loader \
      ${RANCHER_PREFIX}-senzing-stream-loader
    ```

### Install senzing-api-server

1. Example:

    ```console
    rancher app install \
      --answers ${RANCHER_ANSWERS_DIR}/senzing-api-server-mysql.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-senzing-api-server \
      ${RANCHER_PREFIX}-senzing-api-server
    ```

1. Port forward to local machine.  Run in a separate terminal window. Example:

    ```console
    export RANCHER_PREFIX=my-senzing-mysql
    export RANCHER_NAMESPACE_NAME=${RANCHER_PREFIX}-namespace

    rancher kubectl port-forward --namespace ${RANCHER_NAMESPACE_NAME} svc/${RANCHER_PREFIX}-senzing-api-server 8889:80
    ```

### Test Senzing REST API server

*Note:* port 8889 on the localhost has been mapped to port 80 in the docker container.
See `rancher kubectl port-forward ...` above.

1. Example:

    ```console
    export SENZING_API_SERVICE=http://localhost:8889

    curl -X GET ${SENZING_API_SERVICE}/heartbeat
    curl -X GET ${SENZING_API_SERVICE}/license
    curl -X GET ${SENZING_API_SERVICE}/entities/1
    ```

## Cleanup

### Switch context for delete

1. Example:

    ```console
    rancher context switch \
      ${RANCHER_PROJECT_NAME}
    ```

### Delete everything in project

1. Example:

    ```console
    rancher app delete ${RANCHER_PREFIX}-senzing-api-server
    rancher app delete ${RANCHER_PREFIX}-senzing-stream-loader
    rancher app delete ${RANCHER_PREFIX}-senzing-mock-data-generator
    rancher app delete ${RANCHER_PREFIX}-phpmyadmin
    rancher app delete ${RANCHER_PREFIX}-mysql-client
    rancher app delete ${RANCHER_PREFIX}-mysql
    rancher app delete ${RANCHER_PREFIX}-kafka-test-client
    rancher app delete ${RANCHER_PREFIX}-kafka
    rancher kubectl delete -f ${GIT_REPOSITORY_DIR}/kubernetes/persistent-volume-claim-opt-senzing.yaml
    rancher kubectl delete -f ${GIT_REPOSITORY_DIR}/kubernetes/persistent-volume-opt-senzing.yaml
    rancher namespace delete ${RANCHER_NAMESPACE_NAME}
    rancher projects delete ${RANCHER_PROJECT_NAME}
    ```  

### Default context after cleanup

1. Switch context.  Example:

    ```console
    rancher context switch Default
    ```

### Delete catalogs

1. Delete Senzing catalog. Example:

    ```console
    rancher catalog delete senzing
    ```
