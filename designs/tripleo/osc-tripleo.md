# OSC Integration with TripleO
In order for OSC to be installable and configurable by [TripleO](#tripleo-home) a TripleO-Heat-Template (THT) must be created. This document outlines the details of this template highlighting OSC specific configuration and environment variables, TripleO required configuration, data persistence, TLS and high availability (HA) settings.   
> Note: Details for HA and other advanced configuration will be added later. The first draft of this document covers required files, template parameters, environment variables and volumes.  

## Assignees
Emanoel Xavier - https://github.com/emanoelxavier

## Background

[TripleO](#tripleo-home) is the friendly name for “OpenStack on OpenStack”. It is an official OpenStack project with the goal of allowing you to deploy and manage a production cloud onto bare metal hardware using a subset of existing OpenStack components.  The deployed production cloud is called **overcloud** and the infrastructure services used for deployment are called **undercloud** as shown in the picture below. 

![](./images/tripleo-services.png)  
*TripleO Undercloud & Overcloud*  

The intent of this work is to allow OSC to be more easily deployed and configured in a OpenStack cloud by using TripleO. OSC will be deployed in the **controller** node of the **overcloud** as shown below:  
![](./images/osc-tripleo-services.png)  
*OSC In the TripleO Overcloud*  

## Constraints and Assumptions
### OSC Packaged Binaries
TripleO can only deploy OSC through a template if the OSC binaries are available in a public repository. There are multiple paths to achieve this:  
1.  RPM package.  
2.  Docker container image:  
  2.1. On an arbitrary Docker repository; or  
  2.2. Built as a Kolla container in the Kolla repository.  

The source of the OSC binaries is outside of the scope for this document but it assumes option *2.2. Kolla repository* at the moment. For any other choice the OSC THT may need changes and/or additional Puppet manifests might be needed.   

### Puppet TripleO
No changes or new files will be needed in the Puppet TripleO Repository: [https://github.com/openstack/puppet-tripleo](https://github.com/openstack/puppet-tripleo)   

## Design Changes
A THT describes the OpenStack deployment in Heat Orchestration Template YAML files and Puppet manifests. A template for the OSC service will be created and registered with TripleO. The goal is for all the changes described on this section to be upstreamed to the [THT repository](#tripleo-repo)

### OSC THT  

#### Template Location  
The OSC THT will be  `/tripleo-heat-templates/docker/services/osc.yaml`. The resource must be registered in the environment file `tripleo-heat-templates/environments/docker.yaml` as follows:  
```OS::TripleO::Services::OSC: ../docker/services/osc.yaml```  

With these files in place the user can include the OSC service as a resource in the *overcloud* by adding `OS::TripleO::Services::OSC` to the *controller* role during deployment and providing *--environment* files with the values of the parameters defined on the template.   

> Note: By default OSC service will be deployed in all the `controller` nodes. As part of the HA configuration we must address this making OSC a single instance.  

#### Template Content
The file `../docker/services/osc.yaml` starts with the template version and a description, followed a set of parameters, a list of resources and outputs generated by the template. For details on THTs, see the [guide](#tht-guide).  

```yaml  
heat_template_version: queens

description: >
  Open Security Controller: service for orchestration of virtual network security functions.

 parameters:
  #  The paramater EndpointMap is required for all service templates*. It provides the location of the service endpoint, i.e,: IP address and port.  
  EndpointMap:
    default: {}
    description: Mapping of service endpoint -> protocol. Typically set
                 via parameter_defaults in the resource registry.
				 
  # The name of the OSC image in the Kolla repository**.
  OSCDockerImage:
    description: The name of the OSC container image to be deployed.
    type: string

  # Placeholder to exemplify an OSC environment variable.
  OSCEnvVar:
    description: Placeholder/sample parameter for an environment variable used by the OSC container.
    type: string
	
  # The source of the mounted volume OSC needs to persist information.
  OSCVolumeSrc:
    description: The source of the mounted volume OSC needs to persist information.
    type: string
	default: "/var/lib/osc/data"
	
  # The parameter contains the path where OSC db password file resides.
  OSCKeyStoreFile:
    default: '/osc-cert/osckeystore.pk12'
    type: string
    description: This contains the DB passwords used in OSC.
    
  # The parameter contains the path where OSC certificate file resides.
  OSCTrustStoreFile:
    default: '/osc-cert/osctruststore.jks' 
    type: string
    description: This contains private key and public certificate for osc access. In addition contains public 
    certificate chain for accessing osc clients (OpenStack and Security Manager).
    
  # The parameter ServiceNetMap is required for service network-isolation i.e. to indicate in which network the service will be hosted.
  ServiceNetMap:
    default: {}
    description: Mapping of service_name -> network name. Typically set
                 via parameter_defaults in the resource registry.  This
                 mapping overrides those in ServiceNetMapDefaults.

resources:
  # This resource containers a static list of configurations and scripts necessary 
  # for the deployment of container services in the *overcloud*
  ContainersCommon:
    type: ./containers-common.yaml

outputs:
  # Service templates must output a role_data value containing the config_settings for the service configuration*.
  role_data:
    description: Role data for the OSC service.
    value:
      config_settings:
        docker_config:
        step_1:
          osc:
            # The name of the OSC image in the Kolla container repo**.
            image: &osc_image {get_param: OSCDockerImage}
            # Docker run parameters for OSC***.
            privileged: false
            net: host # Uses the host network
            detach: false
            user: root
            restart: always
            volumes:
              list_concat:
                - {get_attr: [ContainersCommon, volumes]}
                -
                  - {get_param: OSCVolumeSrc}:/opt/vmidc/bin/data/
            environment:
              - OSC_ENV_VAR={get_param: OSCEnvVar}
              - TripleO::Services::OSC::tls_truststore_file: {get_param:OSCTrustStoreFile}
              - TripleO::Services::OSC::keystore_file: {get_param:OSCKeyStoreFile}
        tripleo.osc.firewall_rules:
              'osc-ui':
                dport:
                  -  {get_param: [EndpointMap, OSCUIPublic, port]}
                  - 443
              'osc-api':
                dport:
                  -  {get_param: [EndpointMap, OSCAPIPublic, port]}
                  - 8090
```

*[THT Tutorial](#tht-tutorial)  

**[Kolla Container Repos](#kolla-container-repos)

***[Docker Run References](#docker-run)

> **Assumption:** The images in the Kolla Repos will be picked and replicated in the TripleO Repo https://hub.docker.com/r/tripleoupstream 

#### OSC Environment Variables  
Environment variables required by OSC can be provided by THT parameters as shown `OSC_ENV_VAR={get_param: OSCEnvVar}`.  
TODO: Replace placeholder with actual environment variables required by OSC.  

#### OSC Mounted Volume  
OSC requires a mounted volume for persistence of data such us the OSC H2 database files, plugins, VNF images, etc. The source of the mount will be the host folder `/var/lib/osc/data` and target `/opt/vmidc/bin/data/`. 

#### OSC User  
The commands executed within the OSC container will done with the *root* user as defined here `user: root`.
> Assumption: The THT community does not have reservations with respect to using the root user within the container. Other OOO container services use the same approach, for instance [nova](https://github.com/openstack/tripleo-heat-templates/blob/master/docker/services/nova-api.yaml) or [aodh](https://github.com/openstack/tripleo-heat-templates/blob/master/docker/services/aodh-api.yaml). The root user is not a strong requirement for OSC but understanding the full impact of changing that we will require further investigation. 

### OSC Service Ports
The OSC service makes use of two ports to expose its functionality: the OSC APIs use the port `8090` and the UI `443` within its container. Those ports must be mapped to the host ports dedicated to OSC. When running the OSC container on Docker these bindings are provided through the Docker run parameter `-p`, for details see [Docker networking](https://docs.docker.com/config/containers/container-networking/). For a OOO deployment, the following configuration is assumed to provide the same effect:
```yaml
tripleo.osc.firewall_rules:
              'osc-ui':
                dport:
                  -  {get_param: [EndpointMap, OSCUIPublic, port]}
                  - 443
              'osc-api':
                dport:
                  -  {get_param: [EndpointMap, OSCAPIPublic, port]}
                  - 8090
```
> **Assumption:** The Docker run param `-p` effectively creates a firewall rule which maps the container port to a port on the Docker host. Unfortunately, the [THT Containers documentation](http://tripleo.org/install/containers_deployment/architecture.html) does not make any reference to port binding. Looking at other existing THT services the closest way to configure this is with an entry similar to above but this semantics and syntax must be validated with a THT/RH engineer.

The value for the endpoint ports `OSCUIPublic` and `OSCAPIPublic` must be defined in the file `network/endpoints/endpoint_data.yaml` for OSC as shown below:
```yaml
        OSCUI:
            Public:
                net_param: Public
            port: 8082
        OSCAPI:
            Public:
                net_param: Public
            port: 8083
```

> **TODO**: The default values `8082` and `8083` are just incremental values based on the existing last entry in the file `network/endpoints/endpoint_data.yaml` we must check with RH whether this is the right way to define these values.

#### Service Upgrade  
TODO: Is there any script we need to add to handle service upgrades?

#### OSC HA Configuration  
TODO: Things to consider:  

1. Configure single instance for OSC
2. Persistence in case of container migration

#### OSC TLS Certificate sourcing
TODO: Is it fine to define environomental variables like TripleO::Services::OSC::tls_truststore_file? As defining as 
above would make it available in puppet scripts.


#### OSC TLS & Secret Data Configuration  
1. Other secret data and passwords  

2. The TLS certificates files are stored in a specific directory and same is indicated over the param section of template.Also this path is exposed as an environoment variable, so that this information is available outside of the heat template definition.


>>Assumption: The system administrator before deploying OSC service shall ensure that TLS Certificates are stored in the 
above mentioned path in the undercloud node.During the deployment, the TLS certificates shall be created on the same 
path/directory on overcloud node and the TLS certificate is copied to this location.


>>Questions: 

>>a. It is not clear will $tls_truststore_file info. is passed as part of Docker RUN command parameters? if not, what additional configurations need to be added to make it pass to Docker RUN command parameters?

>>b. In the above declaration we are assuming that $tls_truststore_file info is defined as part of class definition and available in the manifest file. Is it possible to define the logic in manifest file, to copy the certificates to another location? otherwise can we add this logic in the resource definition section in heat template to have this copy logic? what is preferred?

>>c. Do we need to hard code the target path for storing the certificate? If so how is this used by puppet?

#### OSC Network Configuration
1. Add two entries in the file `network/service_net_map.j2.yaml` under the `ServiceNetMapDefaults` section for `OSCUINetwork` and `OSCAPINetwork` like below. This is required to prevent OSC service to be part of provisioning network.
    ```yaml
        ServiceNetMapDefaults:
            default:
                ...
                OSCUINetwork: internal_api
                OSCAPINetwork: internal_api
    ```
    >Assumption: OSC Service needs to be placed under internal_api network by default.

2. Add the port entries in the file `network/endpoints/endpoint_data.yaml` as mentioned in [OSC Service Ports](#osc-service-ports).

## Tests
TBD

## References
### [TripleO Home](http://tripleo.org/)  
### [TripleO Repo](https://github.com/openstack/tripleo-heat-templates)  
### [THT Guide](https://docs.openstack.org/heat/pike/template_guide/hot_guide.html)  
### [THT Tutorial](http://tripleo.org/install/developer/tht_walkthrough/changes-tht.html)  
### [Kolla Container Repos](https://hub.docker.com/r/kolla/centos-binary-neutron-server-opendaylight)  
### [Docker Run](https://docs.docker.com/engine/reference/run/)
### [TLS Certificate](https://github.com/openstack/tripleo-heat-templates/blob/master/environments/ssl/enable-tls.yaml)
