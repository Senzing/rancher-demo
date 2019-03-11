# rancher-db2-cluster-demo

## Overview

The following diagram shows the relationship of the Rancher apps, docker containers, and code in this Rancher demonstration.

![Image of architecture](docs/img-architecture/architecture.png)

### Contents

1. [Demonstrate](#demonstrate)
    1. [Set environment variables for repository](#set-environment-variables-for-repository)
    1. [Clone repository](#clone-repository)
    1. [Prerequisites](#prerequisites)
    1. [Set environment variables](#set-environment-variables)
    1. [Create custom answer files](#create-custom-answer-files)
    1. [Set default context](#set-default-context)
    1. [Add catalog](#add-catalog)
    1. [Create project](#create-project)
    1. [Switch context](#switch-context)
    1. [Create namespace](#create-namespace)
    1. [Create persistent volume](#create-persistent-volume)
    1. [Install Kafka](#install-kafka)
    1. [Install Kafka test client](#install-kafka-test-client)
    1. [Install mySQL](#install-mysql)
    1. [Install phpMyAdmin](#install-phpmyadmin)
    1. [Test access to senzing docker images](#test-access-to-senzing-docker-images)
    1. [Install mock-data-generator](#install-mock-data-generator)
    1. [Install stream-loader](#install-stream-loader)
    1. [Install senzing-api-server](#install-senzing-api-server)
    1. [Test Senzing REST API server](#test-senzing-rest-api-server)
1. [Cleanup](#cleanup)
    1. [Switch context for delete](#switch-context-for-delete)
    1. [Delete everything in project](#delete-everything-in-project)
    1. [Default context after cleanup](#default-context-after-cleanup)
    1. [Delete catalog](#delete-catalog)

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

1. Because an independent download is needed for the DB2 ODBC client, the
   [senzing/python-db2-cluster-base](https://github.com/Senzing/docker-python-db2-cluster-base)
   docker image must be manually built.
   Follow the build instructions at
   [github.com/Senzing/docker-python-db2-cluster-base](https://github.com/Senzing/docker-python-db2-cluster-base#build)

1. Verify `senzing/python-db2-cluster-base` is a local image.

    ```console
    sudo docker images | grep "senzing/python-db2-cluster-base"
    ```

1. Make Senzing docker images.

    ```console
    sudo docker build --tag senzing/db2express-c https://github.com/senzing/docker-db2express-c.git
    sudo docker build --tag senzing/db2 https://github.com/senzing/docker-db2.git
    sudo docker build --tag senzing/hello-world https://github.com/senzing/docker-hello-world.git
    sudo docker build --tag senzing/mock-data-generator https://github.com/senzing/mock-data-generator.git

    sudo docker build \
       --tag senzing/stream-loader-for-db2-cluster \
       --build-arg BASE_IMAGE=senzing/python-db2-cluster-base \
       https://github.com/senzing/stream-loader.git
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
      "db2" \
      "db2express-c" \
      "hello-world" \
      "mock-data-generator" \
      "senzing-api-server" \
      "stream-loader-for-db2-cluster"; \
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
    export RANCHER_PREFIX=my
    ```

1. Set environment variables listed in "[Set environment variables for repository](#set-environment-variables-for-repository)".
1. Environment variables used in `rancher` CLI commands.

    ```console
    export RANCHER_PROJECT_NAME=${RANCHER_PREFIX}-project-1
    export RANCHER_NAMESPACE_NAME=${RANCHER_PREFIX}-namespace-1
    ```

### Create custom answer files

1. Copy example answer files. Example:

    ```console
    mkdir ${GIT_REPOSITORY_DIR}/rancher-answers
    cp ${GIT_REPOSITORY_DIR}/rancher-answer-examples/*.yaml ${GIT_REPOSITORY_DIR}/rancher-answers
    ````

1. Modify ${GIT_REPOSITORY_DIR}/rancher-answers/hello-world.yaml
    1. **image.repository**
        1. Template: "${DOCKER_REGISTRY_URL}/senzing/hello-world"
        1. Example: `'image.repository': "my.docker-registry.com:5000/senzing/hello-world"`  
1. Modify ${GIT_REPOSITORY_DIR}/rancher-answers/mock-data-generator.yaml
    1. **image.repository**
        1. Template: "${DOCKER_REGISTRY_URL}/senzing/mock-data-generator"
        1. Example: `'image.repository': "my.docker-registry.com:5000/senzing/mock-data-generator"`
    1. **senzing.kafkaBootstrapServerHost**
        1. Use hostname of your Kafka server.
1. Modify ${GIT_REPOSITORY_DIR}/rancher-answers/phpmyadmin.yaml
    1. **db.host**
        1. Use hostname of your mySQL server.
1. Modify ${GIT_REPOSITORY_DIR}/rancher-answers/senzing-api-server.yaml
    1. **image.repository**
        1. Template: "${DOCKER_REGISTRY_URL}/senzing/senzing-api-server"
        1. Example: `'image.repository': "my.docker-registry.com:5000/senzing/senzing-api-server"`
1. Modify ${GIT_REPOSITORY_DIR}/rancher-answers/stream-loader.yaml
    1. **image.repository**
        1. Template: "${DOCKER_REGISTRY_URL}/senzing/mock-data-generator"
        1. Example: `'image.repository': "my.docker-registry.com:5000/senzing/stream-loader"`
    1. **senzing.databaseUrl**
        1. Template:  "mysql://g2:g2@${MYSQL_HOSTNAME}:3306/G2"
        1. Example: `mysql://g2:g2@my.sql-server.com:3306/G2`
    1. **senzing.kafkaBootstrapServerHost**
        1. Use hostname of your Kafka server.

### Set default context

1. Switch context.  Example:

    ```console
    rancher context switch \
      Default
    ```

### Add catalog

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

1. Copy example kubernetes files. Example:

    ```console
    mkdir ${GIT_REPOSITORY_DIR}/kubernetes
    cp ${GIT_REPOSITORY_DIR}/kubernetes-examples/*.yaml ${GIT_REPOSITORY_DIR}/kubernetes
    ````

1. Modify ${GIT_REPOSITORY_DIR}/kubernetes/persistent-volume-claim-db2-data-stor.yaml
    1. **namespace**
        1. Template: "${RANCHER_PREFIX}-namespace-1"
        1. Example: `namespace: mytest-namespace-1`

1. Create "persistent volume" for `/opt/senzing` directory. Example:

    ```console
    rancher kubectl create \
      -f ${GIT_REPOSITORY_DIR}/kubernetes/persistent-volume-db2-data-stor.yaml
    ```

1. Create "persistent volume claim" for `/opt/senzing` directory. Example:

    ```console
    rancher kubectl create \
      -f ${GIT_REPOSITORY_DIR}/kubernetes/persistent-volume-claim-db2-data-stor.yaml
    ```

### Install Kafka

1. Example:

    ```console
    rancher app install \
      --answers ${GIT_REPOSITORY_DIR}/rancher-answers/kafka.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      library-kafka \
      ${RANCHER_PREFIX}-kafka
    ```

### Install Kafka test client

1. Install Kafka test client app. Example:

    ```console
    rancher app install \
      --answers ${GIT_REPOSITORY_DIR}/rancher-answers/kafka-test-client.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-kafka-test-client \
      ${RANCHER_PREFIX}-kafka-test-client
    ```

1. Run the test client. Run in a separate terminal window. Example:

    ```console
    export RANCHER_PREFIX=my
    export RANCHER_NAMESPACE_NAME=${RANCHER_PREFIX}-namespace-1

    rancher kubectl exec \
      -it \
      -n ${RANCHER_NAMESPACE_NAME} \
      ${RANCHER_PREFIX}-kafka-test-client -- /usr/bin/kafka-console-consumer \
        --bootstrap-server ${RANCHER_PREFIX}-kafka-kafka:9092 \
        --topic senzing-kafka-topic \
        --from-beginning
    ```

### Install DB2

1. Variables for the secret. Example:

    ```console
    export DOCKER_SECRET=my-hub-docker-login

    export DOCKER_USERNAME=myusername
    export DOCKER_PASSWORD=mypassword
    export DOCKER_EMAIL=me@example.com
    ```

1. Create secret.

    ```console
    rancher kubectl create secret docker-registry ${DOCKER_SECRET} \
      --docker-username=${DOCKER_USERNAME} \
      --docker-password=${DOCKER_PASSWORD} \
      --docker-email=${DOCKER_EMAIL} \
      --namespace=${RANCHER_NAMESPACE_NAME}
    ```

1. Patch default serviceaccount.

    ```console
    rancher kubectl patch serviceaccount default \
      -p ‘{“imagePullSecrets”: [{“name”: “${DOCKER_SECRET}”}]}’ \
      --namespace=${RANCHER_NAMESPACE_NAME}
    ```

1. Launch apps.  Example:

    CORE database

    ```console
    rancher app install \
      --answers ${GIT_REPOSITORY_DIR}/rancher-answers/ibm-db2oltp-dev-core.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      ibm-db2oltp-core \
      ${RANCHER_PREFIX}-ibm-db2oltp-core
    ```

    RES database

    ```console
    rancher app install \
      --answers ${GIT_REPOSITORY_DIR}/rancher-answers/ibm-db2oltp-dev-res.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      ibm-db2oltp-res \
      ${RANCHER_PREFIX}-ibm-db2oltp-res
    ```

    LIBFE database

    ```console
    rancher app install \
      --answers ${GIT_REPOSITORY_DIR}/rancher-answers/ibm-db2oltp-dev-libfe.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      ibm-db2oltp-libfe \
      ${RANCHER_PREFIX}-ibm-db2oltp-libfe
    ```

### Initialize DB2


1. Create database on "CORE" DB2 server. In docker container, run

    
    ```console
    sudo su - db2inst1    
    export DB2_NODE_NAME=G2_CORE
    cd ~
    
    curl -X GET http://forthdimension.com/senzing/g2core-schema-db2-create.sql -o g2core-schema-db2-create.sql

    db2 catalog tcpip node ${DB2_NODE_NAME} remote ${HOSTNAME} server 50000
    db2 attach to ${DB2_NODE_NAME} user db2inst1 using db2inst1
    db2 create database G2 using codeset utf-8 territory us
    db2 connect to G2 user db2inst1 using db2inst1
    db2 list tables
    db2 -tf g2core-schema-db2-create.sql | tee /tmp/g2schema.out
    db2 list tables
    db2 connect reset
    ```

1. Create database on "RES" DB2 server. In docker container, run

    ```console
    sudo su - db2inst1    
    export DB2_NODE_NAME=G2_RES
    cd ~
    
    curl -X GET http://forthdimension.com/senzing/g2core-schema-db2-create.sql -o g2core-schema-db2-create.sql

    db2 catalog tcpip node ${DB2_NODE_NAME} remote ${HOSTNAME} server 50000
    db2 attach to ${DB2_NODE_NAME} user db2inst1 using db2inst1
    db2 create database G2 using codeset utf-8 territory us
    db2 connect to G2 user db2inst1 using db2inst1
    db2 list tables
    db2 -tf g2core-schema-db2-create.sql | tee /tmp/g2schema.out
    db2 list tables
    db2 connect reset
    ```

1. Create database on "LIBFE" DB2 server. In docker container, run

    ```console
    sudo su - db2inst1    
    export DB2_NODE_NAME=G2_LIBFE
    cd ~
    
    curl -X GET http://forthdimension.com/senzing/g2core-schema-db2-create.sql -o g2core-schema-db2-create.sql

    db2 catalog tcpip node ${DB2_NODE_NAME} remote ${HOSTNAME} server 50000
    db2 attach to ${DB2_NODE_NAME} user db2inst1 using db2inst1
    db2 create database G2 using codeset utf-8 territory us
    db2 connect to G2 user db2inst1 using db2inst1
    db2 list tables
    db2 -tf g2core-schema-db2-create.sql | tee /tmp/g2schema.out
    db2 list tables
    db2 connect reset
    ```

### Install mock-data-generator

1. Example:

    ```console
    rancher app install \
      --answers ${GIT_REPOSITORY_DIR}/rancher-answers/mock-data-generator.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-mock-data-generator \
      ${RANCHER_PREFIX}-senzing-mock-data-generator
    ```

### Install stream-loader

1. Example:

    ```console
    rancher app install \
      --answers ${GIT_REPOSITORY_DIR}/rancher-answers/stream-loader.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-stream-loader \
      ${RANCHER_PREFIX}-senzing-stream-loader
    ```

### Install senzing-api-server

1. Example:

    ```console
    rancher app install \
      --answers ${GIT_REPOSITORY_DIR}/rancher-answers/senzing-api-server.yaml \
      --namespace ${RANCHER_NAMESPACE_NAME} \
      senzing-api-server \
      ${RANCHER_PREFIX}-senzing-api-server
    ```

1. Port forward to local machine.  Run in a separate terminal window. Example:

    ```console
    export RANCHER_PREFIX=my
    export RANCHER_NAMESPACE_NAME=${RANCHER_PREFIX}-namespace-1

    rancher kubectl port-forward --namespace ${RANCHER_NAMESPACE_NAME} svc/my-senzing-api-server 8889:8080
    ````

### Test Senzing REST API server

*Note:* port 8889 on the localhost has been mapped to port 8080 in the docker container.
See `rancher kubectl port-forward ...` above.

1. Example:

    ```console
    export SENZING_API_SERVICE=http://localhost:8889

    curl -X GET ${SENZING_API_SERVICE}/heartbeat
    curl -X GET ${SENZING_API_SERVICE}/license
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

    rancher app delete ${RANCHER_PREFIX}-ibm-db2oltp-libfe
    rancher app delete ${RANCHER_PREFIX}-ibm-db2oltp-res
    rancher app delete ${RANCHER_PREFIX}-ibm-db2oltp-core
    rancher app delete ${RANCHER_PREFIX}-kafka-test-client
    rancher app delete ${RANCHER_PREFIX}-kafka
    rancher kubectl delete -f ${GIT_REPOSITORY_DIR}/kubernetes/persistent-volume-claim-db2-data-stor.yaml
    rancher kubectl delete -f ${GIT_REPOSITORY_DIR}/kubernetes/persistent-volume-db2-data-stor.yaml
    rancher namespace delete ${RANCHER_NAMESPACE_NAME}
    rancher projects delete ${RANCHER_PROJECT_NAME}
    ```  

### Default context after cleanup

1. Switch context.  Example:

    ```console
    rancher context switch Default
    ```

### Delete catalog

1. Delete Senzing catalog. Example:

    ```console
    rancher catalog delete senzing
    ```
