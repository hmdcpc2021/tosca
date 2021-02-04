# Developing and Deploying an Algorithm

An attempt to separate the roles of algorithm development and algorithm deployment.

## Algorithm Development

### Containers

An algorithm developer creates and algorithm, which consists of **one or more** containers. The developer should be able to provide the following information for each container, with the requirements in bold:

| description                                                    | key                            | value (type)                 |
| ---------------------------------------------------------------| ------------------------------ |:----------------------------:|
| **A name (hostname) for the container**                        | **name**                       | **string**                   |
| **The type of the container (see Container Types section)**    | **type**                       | **string**                   |
| **Full URI of container image**                                | **image**                      | **string**                   |
| **Abstract Host (VM) (see Virtual Machines section)**          | **host**                       | **string**                   |
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
| Please see the relevant section of the [Kubernetes API](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-requests-and-limits-of-pod-and-container) for values   | resources                      | map[`limits`\|`requests`]... |

##### Table 1. Required (in bold) and optional metadata for describing containers

#### Container Types

Then the algorithm developer should decide the strategy for scheduling the containers with Kubernetes (since developers are not necessarily Kubernetes experts, most will usually rely on the default "Deployment"). 

The algorithm developer should also decide the appropriate runtime for their container.

** **OPEN QUESTION: Should the developer decide the runtime?**

    A Docker image can be executed by many runtimes (containerd/CRI-O/Singularity can
    all run Docker images) so most containers are "runtime-agnostic". For the sake of
    simplicity lets for now assume the algorithm developer is responsible for selecting
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

##### Table 2. Possible container types and deployment strategies

### Abstract Virtual Machines / Requirements

It is also assumed that the algorithm developer is the best person for deciding what the (hardware) requirements for each container are. As such, they should define the desired hosting infrastructure (**one or more** VMs) and then map each container to a VM. To remain cloud-agnostic, the hosting infrastructure must be defined at a high-level. The following information can be provided for each virtual machine (notice there are no cloud specific IDs etc...):

| description                                                    | key                            | value (type)             |
| ---------------------------------------------------------------| ------------------------------ |:------------------------:|
| The minimum number of CPUs (or vCPUS)                          | num_cpus                       | string                   |
| The minimum desired RAM size                                   | mem_size                       | scalar (MB/GB)           |
| Operating system architecture                                  | os_architecture                | string                   |
| Operating system type (linux/windows)                          | os_type                        | string                   |
| Operating system distribution (ubuntu/fedora)                  | os_distribution                | string                   |
| Operating system version                                       | os_version                     | string                   |
| Ports to open (at cloud-level)                                 | ports                          | []int                    |

##### Table 3. Abstract Virtual Machine metadata (Requirements)

## Algorithm Deployment

The algorithm developer's work is now done. They have provided a description of their container(s) and a description of the infrastructure (from a high-level). Now the work passes to a new person who will be responsible for deploying the algorithm to a specific cloud - for now, lets call them the **cloud operator** and assume that they have an account with one or more clouds that they intend to use for deployment. Now, the requirements that were specified by the algorithm developer must be matched with virtual machines that will be provisioned on the desired cloud(s).

### Matching Abstract Requirements to Concrete Virtual Machines

I think the best way to do this is for the cloud operator to provide a document (a special TOSCA ADT) that will describe the possible virtual machines that can be created on their cloud(s). Each node in this document will be a ready-to-provision Virtual Machine, and will therefore need all the account specific IDs that describe the desired VM (they can reference Table 4 and 5 at the end of this README). Each Virtual Machine node should also include plain english descriptions of its hardware capabilities.

*we can define and match any requirements using this approach - GPUs or SGX for example, or even non-hardware requirements (like cost or data center location)*

The requirements defined by the algorithm developer (abstract VM metadata) can now be matched to the plain english descriptions provided by the cloud operator, and we can select the most appropriate concrete VM at deployment time (this needs to be implemented in MiCADO). Here is an example:

**ADT prepared by the Algorithm Developer**
```yaml
tosca_version: tosca_simple_profile_1_2
description: example ADT describing an algorithm
imports: github.com/micado-scale/tosca/micado_types.yaml

topology_template:
  node_templates:
    algorithm_main_container:
      type: MiCADO.Container.Application.Docker
      properties:
        image: myrepo/myalgorithm:v0.2.0
      requirements:
        - host: app_vm

    algorithm_db_container:
      type: MiCADO.Container.Application.Docker
      properties:
        image: mongodb:1.0
      requirements:
        - host: db_vm

    app_vm:
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

    db_vm:
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

**Document of available VMs prepared by the Cloud Operator**
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

The concrete ADT selects the best match VMs from the document prepared by the cloud operator. With a solution like this, each person very clearly only provides the metadata that is required of them.

**Below are the tables a cloud operator will reference to create their "available VMs document"**

#### OpenStack Nova

The table below shows the metadata available for defining an **OpenStack Nova** Compute Node. The **required** keys
are shown in Table 4 in **bold**. 

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

##### Table 4. Required (in bold) and optional metadata for describing OpenStack Nova virtual machines in MiCADO


#### AWS EC2

The table below shows the metadata available for defining an **EC2** instance in AWS. The **required** keys
are shown in Table 5 in **bold**. 

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

##### Table 5. Required (in bold) and optional metadata for describing AWS EC2 virtual machines in MiCADO
