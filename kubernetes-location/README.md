---
section: Kubernetes Location
section_type: inline
section_position: 4
---

### Kubernetes Location

Cloudsoft AMP can deploy applications to [Kubernetes](http://kubernetes.io/) (k8s) clusters both provisioned by Cloudsoft AMP and set up manually.

Here is an example catalog item to add a Kubernetes endpoint to your catalog locations:

    brooklyn.catalog:
      id: my-kubernetes-cluster
      name: "My Kubernetes Cluster"
      itemType: location
      item:
        type: kubernetes
        brooklyn.config:
          endpoint: << endpoint >>
          identity: "guest"
          credential: "guest"
          image: "cloudsoft/centos:7"
          loginUser.password: "p4ssw0rd"

There are a lot of ways to authenticate with kubernetes.  AMP configuration for these are documented in the [reference](kubernetes-location-configuration).  For example to use client
certificates use the following example yaml:

    brooklyn.catalog:
      id: my-kubernetes-cluster
      name: "My Kubernetes Cluster"
      itemType: location
      item:
        type: kubernetes
        brooklyn.config:
          endpoint: << endpoint >>
          caCertData: |
            << Generated Ca Cert (see below) >>
          clientCertData: |
            << Generated Cert (see below) >>
          clientKeyData: |
            << Generated client key (see below) >>
          image: "cloudsoft/centos:7"
          loginUser.password: "p4ssw0rd"

AMP Deploys to a Kubernetes cluster by modelling a `KubernetesPod` entity which is made up of multiple heterogeneous `DockerContainer` entities.

#### Plain-AMP blueprints

Standard AMP blueprints can be deployed within a K8s cluster, here's a simple example:

    location:
      << see above >>

    services:
      - type: org.apache.brooklyn.entity.software.base.VanillaSoftwareProcess
        name: "Simple Netcat Server"
        brooklyn.config:
          provisioning.properties:
            inboundPorts: [ 22, 4321 ]
            env:
              CLOUDSOFT_ROOT_PASSWORD: "p4ssw0rd"
          launch.command: |
            yum install -y nc
            echo hello | nc -l 4321 &
            echo $! > $PID_FILE

For each entity AMP will create a [_Deployment_](http://kubernetes.io/docs/user-guide/deployments/).
This deployment contains a [_ReplicaSet_](http://kubernetes.io/docs/user-guide/replicasets/)
of replicas (defaulting to one) of a [_Pod_](http://kubernetes.io/docs/user-guide/pods/).
Each pod contains a single SSHable container based on the `cloudsoft/centos:7` image.

It will then install and launch the entity. Each `inboundPort` will be exposed as a
[_NodePort_](http://kubernetes.io/docs/user-guide/services/#type-nodeport) in a _Service_.

The config options in the `provisioning.properties` section allow the location to be further customized for each entity, as follows:

- **env**  The `cloudsoft/centos:7` image uses an environment variable named `CLOUDSOFT_ROOT_PASSWORD`
   to assign the SSH login user password. This must match the `loginUser.password` configuration on the location.
- **inboundPorts**  The set of ports that should be exposed by the service.

#### Docker Container based blueprints

Alternatively AMP can launch instances based on a `DockerContainer`, this means additional configuration such as custom docker images can be specified. Here's an example which sets up a [Wordpress](https://wordpress.org/) instance:

    location:
      << see above >>

    services:
    - type: org.apache.brooklyn.container.entity.kubernetes.KubernetesPod
      brooklyn.children:
      - type: org.apache.brooklyn.container.entity.docker.DockerContainer
        id: wordpress-mysql
        name: "MySQL"
        brooklyn.config:
          docker.container.imageName: mysql:5.6
          docker.container.inboundPorts: [ "3306" ]
          docker.container.environment:
            MYSQL_ROOT_PASSWORD: "password"
          provisioning.properties:
            deployment: wordpress-mysql
      - type: org.apache.brooklyn.container.entity.docker.DockerContainer
        id: wordpress
        name: "Wordpress"
        brooklyn.config:
          docker.container.imageName: wordpress:4-apache
          docker.container.inboundPorts: [ "80" ]
          docker.container.environment:
            WORDPRESS_DB_HOST: "wordpress-mysql"
            WORDPRESS_DB_PASSWORD: "password"

The `DockerContainer` entities each create their own _DeploymentConfig_, _ReplicationController_ and _Pod_ entities,
in the same way as the standard AMP blueprint entities above. Each container entity can be further configured using the following options:

- **docker.container.imageName** The Docker image to use for the container
- **docker.container.inboundPorts** The set of ports on the container that should be exposed
- **docker.container.environment** A map of environment variables for the container

Note the use of **deployment** in the `provisioning.properties` configuration, to set the hostname of the MySQL container to allow the Wordpress Apache server to connect to it.

#### Kubernetes location configuration

To configure the kubernetes location for different kubernetes setups the following configuration params are available.

- **caCertData** Data for CA certificate
- **caCertFile** URL of resource containing CA certificate data
- **clientCertData** Data for client certificate
- **clientCertFile** URL of resource containing client certificate data
- **clientKeyData** Data for client key
- **clientKeyFile** URL of resource containing client key data
- **clientKeyAlgo** Algorithm used for the client key
- **clientKeyPassphrase** Passphrase used for the client key
- **oauthToken** The OAuth token data for the current user
- **namespace** Namespace where resources will live; the default is 'amp'
- **namespace.create** Whether to create the namespace if it does not exist
  - **default** true
- **namespace.deleteEmpty** Whether to delete an empty namespace when releasing resources
  - **default** false
- **persistentVolumes** Set up persistent volumes.
- **deployment** The name of the service the deployment will use.
- **image** Docker image to be deployed into the pod
- **osFamily** OS family, e.g. CentOS, Ubuntu
- **osVersionRegex** Regular expression for the OS version to load
- **env** Environment variables to inject when starting a container
- **replicas** Number of replicas of the pod
  - **default** 1
- **secrets** Kubernetes secrets to be added to the pod
- **limits** Kubernetes resource limits
- **privileged** Whether Kubernetes should allow privileged containers
  - **default** false
- **loginUser** Override the user who logs in initially to perform setup
  - **default** root
- **loginUser.password** Custom password for the user who logs in initially
- **injectLoginCredential** Whether to inject login credentials (if null, will infer from image choice); ignored if explicit 'loginUser.password' supplied
