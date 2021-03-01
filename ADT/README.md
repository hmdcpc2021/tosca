# Developing and Deploying an Algorithm
Separating microservices, algorithms and infrastructure

An algorithm is created by an Algorithm Provider and consists of **one or more** microservices.
Before an algorithm can be created, the various microservices that make up that algorithm
must be published. Published microservices can then be combined to create an algorithm. An algorithm
can later be paired with data and model to create a deployable unit called a DMA tuple. DMA tuples
can then be deployed on some specific cloud infrastructure.

## 1. Publishing Microservices

### 1.1 Microservices Configuration Details

Microservices must be published before they can be used to create an algorithm. Currently,
microservices are assumed to be containerised applications. A microservice has configuration
details, as well as high-level descriptive metadata. The configuration details for a microservice
can be extracted from any one of the following common forms of container deployment:
- The `docker run` command
- The `podman run` command
- Docker-Compose v3
- Kubernetes Manifest
- Helm Chart
- MiCADO Application Description Template

Extracting configuration details from these formats results in the generation of a **Microservice**
**Description Template** in TOSCA. The following fields will be automatically filled, or left in a
default state.

| description                                                    | key                            | value (type)                 |
| ---------------------------------------------------------------| ------------------------------ |:----------------------------:|
| A name (hostname) for the container                            | name                           | string                       |
| The type of the workload (see Workload Types section)          | type                           | string                       |
| Full URI of container image                                    | image                          | string                       |
| Command to run / entrypoint to the container                   | command                        | []string                     |
| Arguments to run command / entrypoint                          | args                           | []string                     |
| Labels to attach to this container                             | labels                         | map[string]string            |
| List of maps for environment variables                         | env                            | []*EnvMap*                   |
| Name of environment variable                                   | *EnvMap*.name                  | string                       |
| Value of environment variable                                  | *EnvMap*.value                 | string                       |
| List of maps for exposed ports                                 | ports                          | [](*Port* or *Service*)      |
| Port to expose on the container IP                             | *Port*.containerPort           | int                          |
| Port to expose on the host interface                           | *Port*.hostPort                | int                          |
| Protocol to use (TCP / UDP)                                    | *Port*.protocol                | string                       |
| Port to expose, reachable by other containers at *NAME*:*PORT* | *Service*.port                 | int                          |
| High-level port to expose on the host (30000-32767)            | *Service*.nodePort             | int                          |
| Port to target on the pods/containers                          | *Service*.targetPort           | int                          |
| Protocol to use (TCP / UDP)                                    | *Service*.protocol             | string                       |
| Any other Workload options defined by the [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/workloads-resources/)   | *any*                      | *any* |

##### Table 1. Generated configuration fields in a Microservice Description Template

#### Workload Types

Based on the format used for generating the configuration details above, a workload type will be
assigned to the Microservice. If the `docker run` or Docker-Compose formats are used, the default
`Deployment` type will be used. If Kubernetes specific formats are used (Manifest or Helm Chart),
then the Kubernetes Workload type will be extracted appropriately.

The microservice publisher can also specify a specific runtime for their container using metadata.

** **OPEN QUESTION: Should the microservice publisher decide the runtime?**

    A Docker image can be executed by many runtimes (containerd/CRI-O/Singularity can
    all run Docker images) so most containers are "runtime-agnostic". For the sake of
    simplicity lets for now assume the microservice publisher is responsible for selecting
    the container runtime too.

| value                                                         |  description                                                      |
| ------------------------------------------------------------- | ----------------------------------------------------------------- |
| MiCADO.Container.Application.Docker                           | Use the Docker runtime                                            |
| MiCADO.Container.Application.containerd                       | Use the containerd runtime                                        |
| MiCADO.Container.Application.CRIO                             | Use the CRI-O runtime                                             |
| MiCADO.Container.Application.Singularity                      | Use the Singularity runtime **(Future-Work)**                     |
| *MiCADO.Container.Application.Docker*.Deployment              | Use the Kubernetes "Deployment" strategy with Docker **(default)**|
| *MiCADO.Container.Application.Docker*.DaemonSet               | "DaemonSet" strategy: Exactly one replica on each VM              |
| *MiCADO.Container.Application.Docker*.StatefulSet             | "StatefulSet" strategy: Stable identifiers and persistent storage |
| *MiCADO.Container.Application.containerd*.Deployment          | "Deployment" strategy with containerd ....                        |
| etc....                                                       |                                                                   |

##### Table 2. Workload types automatically assigned to Microservices

### 1.2 Microservices Hardware/Infrastructure Requirements

The Microservices publisher should also provide metadata to specify what the hardware, infrastructure or other hosting requirements are for each Microservice, in a cloud-agnostic way. The following metadata can be provided for each virtual machine:

| description                                                    | key                            | value (type)             |
| ---------------------------------------------------------------| ------------------------------ |:------------------------:|
| The minimum number of CPUs (or vCPUS)                          | min_cpu                        | int                      |
| The maximum number of CPUs (or vCPUS)                          | max_cpu                        | int                      |
| The minimum desired RAM size                                   | min_mem                        | scalar (MB/GB)           |
| The maximum desired RAM size                                   | max_mem                        | scalar (MB/GB)           |
| Operating system architecture                                  | os_arch                        | string                   |
| Operating system type (linux/windows)                          | os_type                        | string                   |
| Operating system distribution (ubuntu/fedora)                  | os_distro                      | string                   |
| Operating system version                                       | os_version                     | string                   |
| Ports to open (at cloud-level)                                 | ports                          | []int                    |

##### Table 3. Hardware / Infrastructure / Virtual Machine metadata (Requirements)

### 1.3 Higher-level metadata for descriptions/searching/filtering

In addition to the configuration details (1.1) and hardware/infrastructure requirements (1.2), higher-level
metadata will be provided. This metadata can be used to display descriptions to users, and support filtering
or searching for Microservices that have been published to the platform.

  From a Nexus perspective, the MDT (generated from data in 1.1 and 1.2) will be uploaded
  to Nexus as a Nexus Asset. The higher-level metadata (in 1.3) will be the Nexus Metadata
  attached to the Nexus Asset.

## 2. Publishing an Algorithm

### 2.1 List of Microservices

Now that the required Microservices have been published, the Algorithm Provider can
publish the algorithm. To do so, they provide a list of the Microservices that make
up this algorithm. The list of Microservices is used to generate an ADT (Application
Description Template) - the descriptor file used to deploy applications in MiCADO.

| description                                                    | key                            | value (type)             |
| ---------------------------------------------------------------| ------------------------------ |:------------------------:|
| The list of Microservices of the algorithm                     | microservices                  | []string                 |

##### Table 4. Algorithm configuration metadata

### 2.2 Topology

By providing this list of Microservices, a default **topology** will be created automatically.

  The term topology here is used to describe the mapping of microservices
  (containers) to infrastructure (virtual machines)

The **default topology** maps each microservice to a virtual machine described by the
Hardware/Infrastructure requirements (see 1.2) for that microservice. This topology
is expressed in the generated ADT.

#### Overwriting the default topology

It is possible for the Algorithm Provider to overwrite the default topology. For example,
if all microservices should be run on a single virtual machine, instead of each microservice
on its own virtual machine.

| description                                                    | key                            | value (type)             |
| ---------------------------------------------------------------| ------------------------------ |:------------------------:|
| The mapping of microservices to their hosts                    | topology_mapping               | map[string]string        |

##### Table 5. Toplogy configuration metadata

### 2.3 Higher-level metadata for descriptions/searching/filtering

In addition to the list of microservices (2.1) and topology (2.2), higher-level
metadata will be provided. This metadata can be used to display descriptions to users, and support filtering
or searching for Algorithms that have been published to the platform.

  From a Nexus perspective, the ADT (generated from data in 2.1 and 2.2) will be uploaded
  to Nexus as a Nexus Asset. The higher-level metadata (in 2.3) will be the Nexus Metadata
  attached to the Nexus Asset.


## 3. Deploying an Algorithm

Now that the algorithm has been published, the DMA Composer will select data, model and an
alogrithm to create a DMA Tuple. It is assumed that the DMA Composer will have an account
with one or more clouds that they intend to use for deployment. Now, the requirements that
were specified by the microservice publisher and the topology specified by the algorithm
provider must be matched with virtual machines that will be provisioned on the desired cloud(s).

### Matching Abstract Requirements to Concrete Virtual Machines

The DMA Composer will provide metadata that will describe the possible infrastructure (virtual machines) that
can be created on their desired cloud(s). The described virtual machines will be ready-to-provision
and will therefore need all the account specific IDs (see Table 6 and 7 for examples). Each description of
a VM should also include plain english descriptions of its hardware capabilities.

  The metadata provided by the DMA Composer will generate an Infrastructure Description Template (IDT).


| description                                                    | key                            | value (type)             |
| ---------------------------------------------------------------| ------------------------------ |:------------------------:|
| **Specific Cloud Service Provider keys/values (Table 7 & 8)**  | **See Tables 7 & 8**           | **See Tables 7 & 8**     |             
| The number of CPUs (or vCPUS)                                  | num_cpus                       | int                      |
| The RAM size                                                   | mem_size                       | scalar (MB/GB)           |
| Operating system architecture                                  | os_arch                        | string                   |
| Operating system type (linux/windows)                          | os_type                        | string                   |
| Operating system distribution (ubuntu/fedora)                  | os_distro                      | string                   |
| Operating system version                                       | os_version                     | string                   |

##### Table 6. Metadata to be provided by DMA Composer to generate an IDT

The requirements defined by the microservice publisher (see 1.2) can now be matched to the
plain english descriptions provided by the DMA Composer, and we can select the most appropriate
concrete VM (which will include specific CSP keys and values) at deployment time.

## 4. An example

### 4.1 Publishing Microservices

**The following `docker run` command is used for publishing a Microservice**
```bash
$ docker run --name algorithm-main-container -e FLAG_VAR=False myrepo/myalgorithm:v0.2.0 /opt/path/to/app
```

**The following metadata is provided to describe the infrastructure requirements**
```json
{
    "min_cpu": "2",
    "max_cpu": "4",
    "min_mem": "2 GB",
    "max_mem": "4 GB",
    "os_arch": "x86_64",
    "os_distro": "ubuntu",
    "os_version": "18.04"
}
```

**The following MDT is generated from the above docker command and metadata**
```yaml
tosca_version: tosca_simple_profile_1_2
description: example MDT describing a microservice
imports: github.com/micado-scale/tosca/micado_types.yaml

topology_template:
  node_templates:
    algorithm-main-container:
      type: MiCADO.Container.Application.Docker
      properties:
        image: myrepo/myalgorithm:v0.2.0
        env:
          name: FLAG_VAR
          value: False
        args: /opt/path/to/app
      requirements:
        - host: algorithm-main-container-host

    algorithm-main-container-host:
      type: MiCADO.Compute
      directives: [ select ]
      node_filter:
        capabilities:
        - host:
            properties:
              num_cpus: { in_range: [ 2, 4 ] }
              mem_size: { in_range: [ "2 GB", "4 GB" ] }
        - os:
            properties:
              architecture: { equal: x86_64 }
              type: { equal: linux }
              distribution: { equal: ubuntu }
              version: { greater_or_equal: "18.04" }
```

#### Assume another different Microservice is published as above, called *algorithm-db-container*

### 4.2 An algorithm is published using the two Microservices just published

**The following list of Microservices is given**
```json
{
    "microservices": ["algorithm-main-container", "algorithm-db-container"]
}
```

**The following ADT is generated based on that list (and uses the default topology)**
```yaml
tosca_version: tosca_simple_profile_1_2
description: example ADT describing an algorithm
imports: github.com/micado-scale/tosca/micado_types.yaml

topology_template:
  node_templates:
    algorithm-main-container:
      type: MiCADO.Container.Application.Docker
      properties:
        image: myrepo/myalgorithm:v0.2.0
        env:
          name: FLAG_VAR
          value: False
        args: /opt/path/to/app
      requirements:
        - host: algorithm-main-container-host

    algorithm-db-container:
      type: MiCADO.Container.Application.Docker
      properties:
        image: mongodb:1.0
      requirements:
        - host: algorithm-db-container-host

    algorithm-main-container-host:
      type: MiCADO.Compute 
      directives: [ select ]
      node_filter:
        capabilities:
        - host:
            properties:
              num_cpus: { in_range: [ 2, 4 ] }
              mem_size: { in_range: [ "2 GB", "4 GB" ] }
        - os:
            properties:
              architecture: { equal: x86_64 }
              type: { equal: linux }
              distribution: { equal: ubuntu }
              version: { greater_or_equal: "18.04" }

    algorithm-db-container-host:
      type: MiCADO.Compute 
      directives: [ select ]
      node_filter:
        capabilities:
        - host:
            properties:
              num_cpus: { greater_or_equal: 4 }
              mem_size: { greater_or_equal: 16 GB }
        - os:
            properties:
              architecture: { equal: x86_64 }
              type: { equal: linux }
              distribution: { equal: ubuntu }
              version: { greater_or_equal: "20.04" }
```

#### Also consider this alternative

**The same list of Microservices is given, but the topology is overwritten**
```json
{
    "microservices": ["algorithm-main-container", "algorithm-db-container"],
    "topology": {
        "algorithm-main-container": "algorithm-main-container-host",
        "algorithm-db-container": "algorithm-main-container-host"
    }
}
```

**This slightly different ADT is generated based on the metadata**
```yaml
tosca_version: tosca_simple_profile_1_2
description: example ADT describing an algorithm
imports: github.com/micado-scale/tosca/micado_types.yaml

topology_template:
  node_templates:
    algorithm-main-container:
      type: MiCADO.Container.Application.Docker
      properties:
        image: myrepo/myalgorithm:v0.2.0
        env:
          name: FLAG_VAR
          value: False
        args: /opt/path/to/app
      requirements:
        - host: algorithm-main-container-host

    algorithm-db-container:
      type: MiCADO.Container.Application.Docker
      properties:
        image: mongodb:1.0
      requirements:
        - host: algorithm-main-container-host

    algorithm-main-container-host:
      type: MiCADO.Compute 
      directives: [ select ]
      node_filter:
        capabilities:
        - host:
            properties:
              num_cpus: { in_range: [ 2, 4 ] }
              mem_size: { in_range: [ "2 GB", "4 GB" ] }
        - os:
            properties:
              architecture: { equal: x86_64 }
              type: { equal: linux }
              distribution: { equal: ubuntu }
              version: { greater_or_equal: "18.04" }
```

### 4.3 Deployment to specific cloud

**Metadata is provided by the DMA Composer that describes available VMs for deployment**
```json
{
    "nova_small_ubuntu_20": {
        "image_id": "IMAGE_ID_FOR_UBUNTU_20_04",
        "flavor_id": "FLAVOR_ID_FOR_SMALL_INSTANCE",
        "project_id": "PROJECT_ID_FOR_OPENSTACK",
        "ssh_key": "REGISTERED_SSH_KEYNAME",
        "num_cpu": "2",
        "mem_size": "2 GB",
        "os_arch": "x86_64",
        "os_distro": "ubuntu",
        "os_version": "20.04"
    },
    "nova_large_ubuntu_20": {
        "image_id": "IMAGE_ID_FOR_UBUNTU_20_04",
        "flavor_id": "FLAVOR_ID_FOR_LARGE_INSTANCE",
        "project_id": "PROJECT_ID_FOR_OPENSTACK",
        "ssh_key": "REGISTERED_SSH_KEYNAME",
        "num_cpu": "4",
        "mem_size": "16 GB",
        "os_arch": "x86_64",
        "os_distro": "ubuntu",
        "os_version": "20.04"
    }
}
```

**The following IDT is generated**
```yaml
tosca_version: tosca_simple_profile_1_2
description: Available VMs for deployment on our platform
imports: github.com/micado-scale/tosca/micado_types.yaml

topology_template:
  node_templates:
    nova_small_ubuntu_20:
      type: MiCADO.Compute.Nova
      properties:
        image_id: IMAGE_ID_FOR_UBUNTU_20_04
        flavor_id: FLAVOR_ID_FOR_SMALL_INSTANCE
        project_id: PROJECT_ID_FOR_OPENSTACK
        ssh_key: REGISTERED_SSH_KEYNAME
      capabilities:
      - host:
          properties:
            num_cpus: 2
            mem_size: 2 GB
      - os:
          properties:
            architecture: x86_64
            type: linux
            distribution: ubuntu
            version: "20.04"

    nova_large_ubuntu_20:
      type: MiCADO.Compute.Nova
      properties:
        image_id: IMAGE_ID_FOR_UBUNTU_20_04
        flavor_id: FLAVOR_ID_FOR_LARGE_INSTANCE
        project_id: PROJECT_ID_FOR_OPENSTACK
        ssh_key: REGISTERED_SSH_KEYNAME
      capabilities:
      - host:
          properties:
            num_cpus: 4
            mem_size: 16 GB
      - os:
          properties:
            architecture: x86_64
            type: linux
            distribution: ubuntu
            version: "20.04"
```

At Deployment time, MiCADO completes the ADT (generated in 4.2) by selecting the best match
VMs from the IDT (generated in 4.3).

**Below are the tables a cloud operator will reference to create their "available VMs document"**

#### OpenStack Nova

The table below shows the metadata available for defining an **OpenStack Nova** Compute Node. The **required** keys
are shown in Table 7 in **bold**. 

**NOTE** that the required value for the `type` key is given in its description.

| key                            | value (type)                 | description                                                          |
| ------------------------------ |:----------------------------:| -------------------------------------------------------------------- |
| **name**                       | **string**                   | **Name of virtual machine**                                          |
| **type**                       | **string**                   | ***tosca.nodes.MiCADO.Nova.Compute***                                |
| **image_id**                   | **string**                   | **ID of VM drive image (Ubuntu 18.04 & 20.04 supported)**            |
| **project_id**                 | **string**                   | **ID of the project to scope to**                                    |
| **network_id**                 | **string**                   | **ID of the network to connect to**                                  |
| flavor_id                      | string                       | ID of the instance flavor **(required if `flavor_name` not given)**  |
| flavor_name                    | string                       | Name of the instance flavor **(required if `flavor_id` not given)**  |
| key_name                       | string                       | Name of the SSH keypair                                              |
| security_groups                | []string                     | List of security group IDs to apply                                  |
| auth_url                       | string                       | Endpoint of the v3 Identity service                                  |

##### Table 7. Required (in bold) and optional metadata for describing OpenStack Nova virtual machines in MiCADO


#### AWS EC2

The table below shows the metadata available for defining an **EC2** instance in AWS. The **required** keys
are shown in Table 8 in **bold**. 

**NOTE** that the required value for the `type` key is given in its description.

| key                            | value (type)                 | description                                                          |
| ------------------------------ |:----------------------------:| -------------------------------------------------------------------- |
| **name**                       | **string**                   | **Name of virtual machine**                                          |
| **type**                       | **string**                   | ***tosca.nodes.MiCADO.EC2.Compute***                                 |
| **region_name**                | **string**                   | **Name of the AWS region to scope to (eg. us-east-1)**               |
| **image_id**                   | **string**                   | **ID of VM drive image (Ubuntu 18.04 & 20.04 supported)**            |
| **instance_type**              | **string**                   | **Name of the instance type to use (eg. t2.small)**                  |
| key_name                       | string                       | Name of the SSH keypair                                              |
| security_group_ids             | []string                     | List of security group IDs to apply                                  |
| subnet_id                      | string                       | ID of the subnet to join                                             |
| tags                           | map[string]string            | Mapping of additional metadata tags                                  |

##### Table 8. Required (in bold) and optional metadata for describing AWS EC2 virtual machines in MiCADO
