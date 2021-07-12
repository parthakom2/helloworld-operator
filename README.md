# Hello World Ansible Operator

This document provides quick steps to get started on a Hello World Ansible operator and deploy on your OLM supported OpenShift Container Platform. 

## Install Server Pre-reqs
  
  An OpenShift Container Platform v4.6 or later with admin connectivity

## Install Client pre-reqs ( For Linux )

  - Golang 1.16.5

    ```
    wget https://golang.org/dl/go1.16.5.linux-amd64.tar.gz
    tar -zxf go1.16.5.linux-amd64.tar.gz 
    mv go /usr/local/bin
    export PATH=$PATH:/usr/local/bin/go/bin
    ```

  - Operator SDK Version v1.9.0

    ```
    RELEASE_VERSION=v1.9.0
    curl -OJL https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk_linux_amd64
    chmod +x operator-sdk_linux_amd64
    mv operator-sdk_linux_amd64 /usr/local/bin/operator-sdk
    operator-sdk version
    ```

  - Python 3.6

    ```
    yum install python36
    ```

  - ansible v2.11

    ```
    pip install ansible==2.11
    ```

  - ansible-runner [OPTIONAL]

    ```
    pip install ansible-runner
    ```
  - opm

    ```
    wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.7.13/opm-linux-4.7.13.tar.gz
    tar -zxf opm-linux-4.7.13.tar.gz 
    mv opm /usr/local/bin/
    ```

  - podman

    ```
    yum install -y podman
    ```

  - Dev Tools

    ```
    yum groupinstall "Development Tools" -y 
    ```

  - Kubernetes Client tools

    ```
    wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.7.13/openshift-client-linux-4.7.13.tar.gz

    tar -zxf openshift-client-linux-4.7.13.tar.gz
    mv oc /usr/local/bin
    mv kubectl /usr/local/bin
    ```


## Set up your operator repo

 - Create the `helloworld-operator` directory

    ```
    mkdir helloworld-operator
    cd helloworld-operator/
    ```

 - Create API scaffolding

    ```
    operator-sdk init --plugins=ansible --domain example.com

    operator-sdk create api --group world --version v1 --kind HelloWorld --generate-role
    ```

-  Add a sample operator logic to create an nginx deployment

  Update `roles/helloworld/tasks/main.yml` with this ansible task which creates an nginx pod.

    ```
    - name: Create a pod
      k8s:
        state: present
        definition:
          apiVersion: apps/v1 
          kind: Deployment 
          metadata:
            name: "helloworld"
            namespace: "{{ ansible_operator_meta.namespace }}"
            labels:
              app: hello
          spec:
            replicas: "{{ replicas }}"
            selector:
              matchLabels:
                app: hello
            template:
              metadata:
              labels:
                app: hello
              spec: 
              containers:
              - name: nginx
                image: quay.io/bitnami/nginx 
    ```

- Authenticate to your container registry, assuming its `example.com` here

  podman login example.com -u <user> -p <password>


## Use Podman

  ```
  sed -i 's/docker build/podman build/g' Makefile
  sed -i 's/docker push/podman push/g' Makefile
  sed -i 's/--container-tool docker/--container-tool podman/g' Makefile
  ```

## Build operator controller docker image

  ```
  make docker-build docker-push IMG="example.com/helloworld-operator:v0.0.1"
  ```

## Generate OLM manifests

  ```
  make bundle IMG="example.com/helloworld-operator:v0.0.1"
  ```

  Note: This will prompt for the OLM metadata and generate the [Cluster Service Version](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/) file

## Build the OLM Bundle image

  ```
  make bundle-build bundle-push BUNDLE_IMG="example.com/helloworld-operator-bundle:v0.0.1"
  ```

## Build and push the catalog index image

   ```
  make catalog-build catalog-push BUNDLE_IMG="example.com/helloworld-operator-bundle:v0.0.1" CATALOG_IMG="example.com/helloworld-operator-catalog:v0.0.1"
   ```


## Install the operator to your OpenShift Cluster

### Create Global Pull Secret 

  ```
   pull_secret=$(echo -n "<user>:<password>" | base64 -w0)
   oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"example.com":{"auth":"'$pull_secret'"\},|' > /tmp/dockerconfig.json
   oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json
  ```

  In case of OCP versions 4.6 or earlier, the cluster nodes will restart at this point.

### Install the operator in OpenShift

- Create a new project

  ```
  oc new-project helloworld
  ```

- Install the Operator Catalog

  ```
  oc create -f install/catalog-source.yaml 

  oc create -f install/operator-group.yaml 
  oc create -f install/subscription.yaml 
  ```

### Create a HelloWorld custom resource

  ```
  apiVersion: world.example.com/v1
  kind: HelloWorld
  metadata:
    name: helloworld-sample
    namespace: helloworld
  spec:
    replicas: 3
  ```

  At this time you should see the operator reconciling the custom resource to create the nginx pods.
