# ParallelCluster Knowledge Transfer
> [!WARNING]
> The documentation on this page is exclusively meant to assist during the development of the [ParallelCluster Provisioner](../provisioner/) in the future.

AWS ParallelCluster is an open-source cluster management tool that allows any organization to deploy High Performance Computing (HPC) clusters on AWS. The tool provides a high-level abstraction / representation for provisioning resources via CloudFormation in an automated and secure manner. It also supports multiple instance types from AWS EC2 and job schedulers such as SLURM.

In the context of the Blue Brain Project, we have utilized ParallelCluster to deploy a proof-of-concept HPC cluster[^ParallelCluster_GitHub]. This cluster is purposely configured to mimic the hardware configuration of the Blue Brain 5 supercomputer, allowing us to evaluate different aspects of running on AWS such as performance, operational costs, and more. In addition, it provided us with the opportunity to envision how the Blue Brain Project could run in any Cloud service in the future and to evaluate new hardware (e.g., AMD's 4th generation EPYC processors).

We also took the opportunity to investigate how parallel file systems like Lustre FSx could benefit through some novel features such as the Lustre Hierarchical Storage Management (HSM) service. In particular, we configured the filesystem to map two S3 buckets via Data Repository Associations (S3-DRA) and expose the content in Lustre FSx[^S3_DRA]. Other storage technologies, such as EFS, were employed to store the home directories and other related resources.

The purpose of this page is to briefly document and reflect back on the relevant aspects and takeaways from using ParallelCluster on AWS.

[^ParallelCluster_GitHub]: See https://github.com/BlueBrain/aws-parallel-cluster to gain insight about the original code and configurations utilized for the ParallelCluster deployment of the Blue Brain Project.
[^S3_DRA]: An S3-centric architecture can enable cross-region data replication and enhanced Disaster Recovery + Data Lifecycle Management, while keeping the overall operational costs relatively low.

## Technical Insight
In this section, we cover some of the most relevant technical details in regard to the customization, deployment and configuration of the ParallelCluster PoC from the Blue Brain Project. 

### Creating a Custom AMI
Utilizing Amazon Linux 2023 out-of-the-box and manually installing the necessary software for the cluster after it has been deployed is a feasible option, but not the most reliable nor convenient. For instance, on every node bootstraping process, several scripts would have to run in order to install the necessary software and configure the node. This fact delays allocation times, increases the operational costs, and can be prone to errors (e.g., lack of Internet connectivity, software incompatibilities, and more). 

Hence, a more reasonable alternative is to build a custom Amazon Machine Image (AMI) that contains all of the necessary settings, software and operating system configurations required for the compute nodes of the cluster. In such case, the compute nodes will only need to bootstrap to be fully functional and configured from the start.

In this case, AWS ParallelCluster provides the possibility to create custom images that are built relying on the EC2 Image Builder service. We used this method to customize the operating system and software of the compute nodes of our ParallelCluster PoC, avoiding the use of NAT gateways and allowing us to keep the cluster on isolation inside its own private VPC.

Below it can be found our YAML configuration file to build the custom AMI with Amazon Linux 2023:

```Yaml
Build:
  ParentImage: arn:aws:imagebuilder:us-east-1:aws:image/amazon-linux-2023-x86/x.x.x
  UpdateOsPackages:
    Enabled: true

  Components:
    - Type: arn
      Value: arn:aws:imagebuilder:us-east-1:671250183987:component/packages/1.0.0/1
    - Type: arn
      Value: arn:aws:imagebuilder:us-east-1:671250183987:component/singularity-ce/4.2.0/1
    - Type: arn
      Value: arn:aws:imagebuilder:us-east-1:671250183987:component/configure-ami/1.0.0/1

  Iam:
    AdditionalIamPolicies:
      - Policy: arn:aws:iam::671250183987:policy/ParallelCluster_S3_GetObject_RPMs
  
  InstanceType: m5.large
  SubnetId: subnet-076ece71c00742d3c
  SecurityGroupIds:
    - sg-0b5941ba4f4d1a9ff

  Tags:  # Set tags for the resources that build the AMI
    - Key: SBO_Billing
      Value: hpc:parallelcluster

Image:
  Tags:  # Set tags specifically for the AMI
    - Key: SBO_Billing
      Value: hpc:parallelcluster
```

The file is divided in two: a `Build` section, which specifies how to build an image, and an `Image` section, which contains details for when the image has been created. Here are some relevant details from the configuration file above:

- **`ParentImage`:** Represents the base image utilized to create our own AMI. In this case, we request Amazon Linux 2023 and ask the Image Builder to update the OS via `UpdateOsPackages`.
- **`Components`:** Defines the modifications that will be applied to the image, such as installing certain software or changing the operating system defaults. More on this topic in the next subsection.
- **`Iam`:** Allows us to provide specific permissions required to apply some of the components. In the example above, we created an IAM Policy to allow access to a specific S3 bucket.
- **`InstanceType` / `SubnetId` / ...:** Specifies the resource configuration to build the custom AMI with the provided components.
- **`Tags`:** Mostly useful for billing and cost control purposes, it allows us to define specific key-value pairs for the resources allocated during the build process, and for the actual AMI stored in the AWS account.

With the YAML file defined to our needs, we can then build the custom image using the ParallelCluster CLI:

```sh
pcluster build-image --image-configuration ami-config.yaml \
                     --image-id "sbo-parallelcluster-ami-al2023-v2"
```

The Image Builder will then utilize CloudFormation to create the image using the guidelines provided inside the YAML file. We can monitor the build process via the AWS Console or using the CLI:

```sh
$ aws cloudformation describe-stacks --stack-name "sbo-parallelcluster-ami-al2023-v2" | jq -r ".Stacks[].StackStatus"
CREATE_IN_PROGRESS
...  # Wait inside a loop until the CF stack has finished
$ pcluster describe-image --image-id "sbo-parallelcluster-ami-al2023-v2" | jq -r ".imageBuildStatus"
BUILD_COMPLETE  # Check for errors using the status
```

The AMI ID can then be queried for use inside the ParallelCluster configuration file, as explained later in this document:

```sh
$ pcluster describe-image --image-id "sbo-parallelcluster-ami-al2023-v2" | jq -r ".ec2AmiInfo.amiId"
ami-042559ee751d3e522
```

This ID indicates ParallelCluster that we would like to utilize our own custom image in the compute nodes of the cluster, instead of the default Amazon Linux 2023.

> [!NOTE]
> More examples on how to manage the custom AMI creation, alongside how to debug issues or how to destroy it, can be found [here](https://github.com/BlueBrain/aws-parallel-cluster/blob/83607e8/.gitlab-ci.yml#L142-L204).

#### Components / Scripts to customize AMI
A critical piece for building an image is to define the so-called Components that customize it. Image Builder uses the AWS Task Orchestrator and Executor (AWSTOE) to build and test components based on YAML documents. These are essentially scripts to customize and test a custom image.

For instance, one of the Components utilized to customize our ParallelCluster image was purposely designed to install some useful tools (e.g., `htop`, `nodeset`, etc.), as well as the latest AWS CLI to trigger certain S3 operations from within the compute cluster. Here is the YAML file that defines our `packages` component:

```Yaml
name: packages
description: Install tools and AWS CLI for ParallelCluster deployments.
schemaVersion: 1.0
phases:
  - name: build
    steps:
     - name: InstallPackages
       action: ExecuteBash
       inputs:
         commands:
           - |
             #!/bin/bash

             set -euo pipefail

             # Install the tools that we will need in the ParallelCluster
             dnf search htop && dnf install -y htop
             python3 -m pip install ClusterShell

             # Install the latest AWS CLI version
             dnf remove -y awscli
             curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip
             unzip /tmp/awscliv2.zip -d /tmp
             /tmp/aws/install
             rm -rf /tmp/aws*
  - name: test
    steps:
      - name: TestPackages
        action: ExecuteBash
        inputs:
          commands:
            - |
              #!/bin/bash

              set -euo pipefail
              
              which htop && htop --version && \
              which nodeset && nodeset --version && \
              which aws && aws --version

```

As it can be observed, a Component contains a mere definition of the Bash scripts required to build and test the component. In the `build` phase, we run several commands to install the software we would like to have in the custom AMI by default. In the `test` phase, we simply ensure that the software is installed, but more complex evaluations can be designed.

With the YAML defined to our needs, we can use the AWS CLI to create the component:

```sh
aws imagebuilder create-component --name "packages"  \
                                  --semantic-version "1.0.0" \
                                  --platform "Linux" \
                                  --tags "SBO_Billing=hpc:parallelcluster" \
                                  --uri "s3://sboinfrastructureassets/components/packages.yaml"
```

The parameters are self-explanatory and the only catch to remember is to upload the YAML file to an S3 bucket of our choice. We can even use tags for billing and cost control purposes.

> [!NOTE]
> More examples on how to manage a component can be found [here](https://github.com/BlueBrain/aws-parallel-cluster/blob/83607e8/.gitlab-ci.yml#L113-L124). The definition of the rest of the components utilized in the custom AMI can be found [here](https://github.com/BlueBrain/aws-parallel-cluster/tree/83607e8/config/ami/components).

### Defining the Cluster Configuration File
In order to deploy an HPC cluster through AWS ParallelCluster, the tool provides several mechanisms amongst which include the command-line interface (default) or the AWS Console via CloudFormation template. In our case, we relied on the CLI to deploy and manage our proof-of-concept HPC cluster on AWS.

In order to do so, one must define a YAML cluster configuration file, which later will be provided to the CLI to specify how we would like our resources to be deployed. As this file is relatively large, the section is divided in subsections for more detail.

> [!IMPORTANT]
> The full YAML configuration file utilized to deploy our proof-of-concept HPC cluster on AWS can be found [here](https://github.com/BlueBrain/aws-parallel-cluster/blob/83607e8/config/compute-cluster.yaml).

#### Base Configuration

The most essential configuration for our cluster includes the `Region` in which to deploy it, the default `Image` to utilize for the compute nodes, and other settings related to cost control and monitoring:

```YAML
Region: us-east-1
Image:
  Os: alinux2023
  CustomAmi: ami-042559ee751d3e522  # sbo-parallelcluster-ami-al2023-v2
Tags:
  - Key: SBO_Billing
    Value: hpc:parallelcluster
Monitoring:
  Logs:
    CloudWatch:
      RetentionInDays: 14
      DeletionPolicy: Delete
```

The `CustomAmi` specifies the AMI ID of the customized image that we defined in the previous section. The `Tags` contain the default tags for the resources employed in the deployment, including any storage solution (e.g., EFS or Lustre FSx). Lastly, the `Monitoring` section allows us to define what is the retention policy for the logs stored in Amazon CloudWatch, in which case we specify 2 weeks and a `Delete` policy to ensure that the logs are fully deleted when the cluster is also deleted.

#### Head Node Configuration

Similar to a leadership class supercomputer, the head node of the cluster defines the entry point for our HPC cluster. It contains not only the master SSH key assigned to the node, but also any particular IAM policy or post-deploy configuration. Furthermore, its hardware specification is critical to prevent timeouts or job allocation failures:

```YAML
HeadNode:
  InstanceType: t3.medium
  Networking:
    SubnetId: subnet-076ece71c00742d3c # compute
    SecurityGroups:
    - sg-0b5941ba4f4d1a9ff # sbo-poc-compute / hpc
  Ssh:
    KeyName: aws_coreservices
  Iam:
    S3Access:
      - BucketName: sboinfrastructureassets
    AdditionalIamPolicies:
      - Policy: arn:aws:iam::671250183987:policy/ParallelCluster_CloudWatch_TagLogGroup_SLURM
  CustomActions:
    OnNodeConfigured:
      Sequence:
        - Script: s3://sboinfrastructureassets/scripts/setup_users.py
          Args:
            - /sbo/home/resources/users.json
            - /sbo/data
        - Script: s3://sboinfrastructureassets/scripts/setup_slurm.sh
          Args:
            - SBO_Billing=hpc:parallelcluster
        - Script: s3://sboinfrastructureassets/scripts/setup_sshd.sh
        - Script: s3://sboinfrastructureassets/scripts/setup_slurmrestd_service.sh
        - Script: s3://sboinfrastructureassets/scripts/setup_nexus_storage_service.sh
```

From the lines above, the most relevant block is the `OnNodeConfigured` section inside `CustomActions`. The purpose of this section is to allow us to run one or more node configuration scripts during bootstrap[^OnNodeConfigured]. For instance, for simplicity, we run a small script that creates local users with their SSH key assigned to allow access without sharing the master SSH key. We also setup the SLURM Prolog / Epilog job management scripts, the SLURM REST service, and other custom job monitoring customizations. Finally, we define and configure the Nexus Storage Service, essential to register scientific data into the knowledge graph of [Nexus](https://bluebrainnexus.io/).

Note that the master SSH key can be easily created using the AWS CLI and stored in the AWS account. For instance, we can create the `aws_coreservices` as follows:

```sh
aws ec2 create-key-pair --key-name aws_coreservices \
                        --query KeyMaterial \
                        --output text > aws_coreservices.pem
```

[^OnNodeConfigured]: Even though some of these configurations are already part of the AWS Components defined while creating the custom AMI, in this case the goal is to tweak the specific head node to behave the way we expect (i.e., the actual compute nodes share the same custom AMI, but not the configuration).

#### Scheduler Configuration

Given the fact that we would like to mimic the same cluster configuration of our Blue Brain 5 supercomputer, we utilize SLURM as the default job scheduler. The following snippet of code contains the base configuration for SLURM, alongside an example queue / partition from the three defined inside the cluster:

```Yaml
Scheduling:
  Scheduler: slurm
  ScalingStrategy: all-or-nothing
  SlurmSettings:
    EnableMemoryBasedScheduling: true
    CustomSlurmSettingsIncludeFile: s3://sboinfrastructureassets/config/slurm_extras.conf
    Database:
      Uri: hpc-slurm-db.ctydazornca3.us-east-1.rds.amazonaws.com:3306
      UserName: slurm_admin
      PasswordSecretArn: arn:aws:secretsmanager:us-east-1:671250183987:secret:hpc_slurm_db_password-6LNuBy
  SlurmQueues:
  - Name: debug # for testing purposes
    AllocationStrategy: lowest-price
    ComputeResources:
    - Name: t3micro
      Instances:
      - InstanceType: t3.micro
      MinCount: 0
      MaxCount: 8
    Networking:
      SubnetIds:
      - subnet-076ece71c00742d3c # compute
      SecurityGroups:
      - sg-0b5941ba4f4d1a9ff # sbo-poc-compute / hpc
    Iam:
      S3Access:
        - BucketName: sboinfrastructureassets
    CustomActions:
      OnNodeConfigured:
        Script: s3://sboinfrastructureassets/scripts/setup_users.py
        Args:
          - /sbo/home/resources/users.json
    CustomSlurmSettings:
      MaxNodes: 8
      MaxTime: 1-00:00:00
  ...
```

The `ScalingStrategy` is designed to cater the different scaling needs of the cluster, allowing us to select one that meets our specific requirements and constraints. In this case, we chose an `all-or-nothing` strategy, which operates on an all-or-nothing basis, meaning that it either scales up completely or not at all.

On the other hand, the `SlurmSettings` section provide us with the opportunity to customize SLURM to our needs. For instance, we integrate changes into the job management scripts of SLURM and require a `TaskProlog` / `TaskEpilog` scripts alongside the existing `Prolog` / `Epilog` scripts. This is mostly to ensure that the jobs get the correct environment for execution (e.g., `$SHMDIR`, required for running large-scale cellular simulations using NEURON+CoreNEURON). In addition, we include a `Database` configuration to estimate overall operational costs in the cluster.

In regard to the queues / partitions, we specify several configurations that are adapted to the use-cases of the project:

- **`debug` (8 nodes `t3.micro` / 24h-limit):** Meant for testing purposes, this partition provides us with the possibility of evaluating changes in SLURM, the software utilized in the cluster (e.g., Singularity), and more.
- **`prod-mpi` (20 nodes `c7a.48xlarge` + 2 nodes `c5d.24xlarge` / 12h-limit):** Heterogeneous partition purely meant for performance evaluations that contain 4th generation AMD EPYC processors with EFA-enabled for tightly coupled communication. In addition, the partition includes 2 nodes with local NVMe storage for use-cases such as [Functionalizer](https://github.com/BlueBrain/functionalizer).
- **`prod-batch` (16 nodes `m5.8xlarge` + 10 nodes `c7a.48xlarge` / 2h-limit):** Heterogeneous partition purely meant to evaluate the scientific software stack of the Blue Brain Project and compare the performance with the Blue Brain 5 supercomputer. For instance, the `m5.8xlarge` nodes contain 32 cores and 128GB of RAM per node. The architecture is based on Intel's Skylake 8175M with AVX-512 support.

In the example provided above, the most relevant configurations are the `InstanceType` to specify the hardware of each node, the `MinCount` to specify how many nodes can be up-and-running regardless of the jobs allocated, the `MaxCount` to define how large is the partition (e.g., in this case, 8 nodes), and also the `CustomSlurmSettings` to personalize the node and allocation limits.

> [!NOTE]
> For further information, all of the SLURM job management scripts deployed in the proof-of-concept cluster can be found [here](https://github.com/BlueBrain/aws-parallel-cluster/tree/83607e8/scripts/s3).

#### Storage Configuration

With the base configuration, the head node and the compute nodes defined, the last setting required to complete our HPC cluster is the shared storage section:

```Yaml
SharedStorage:
  - Name: Efs-Home
    StorageType: Efs
    MountDir: /sbo/home
    EfsSettings:
      FileSystemId: fs-0c2a2f3ad1b1beeca
  - Name: FsxLustre-Persistent
    StorageType: FsxLustre
    MountDir: /sbo/data
    FsxLustreSettings:
      DeploymentType: PERSISTENT_2
      StorageCapacity: 1200  # 1.2TiB (minimum allowed)
      PerUnitStorageThroughput: 250  # Bandwidth of 250Mbps/TiB
      DataCompressionType: LZ4  # Data compression for higher-throughput
      DataRepositoryAssociations:
        - Name: Containers-DRA
          BatchImportMetaDataOnCreate: true
          DataRepositoryPath: s3://sboinfrastructureassets/containers
          FileSystemPath: /containers
          AutoImportPolicy: [ NEW, CHANGED, DELETED ]
        - Name: Nexus-DRA
          BatchImportMetaDataOnCreate: true
          DataRepositoryPath: s3://sbonexusdata
          FileSystemPath: /project
          AutoExportPolicy: [ NEW, CHANGED, DELETED ]
          AutoImportPolicy: [ NEW, CHANGED, DELETED ]
```

We opted for having a shared EFS designed for home directories and basic configuration scripts (i.e., created externally and provided here via the `FileSystemId`), and a high-performance parallel file system with Lustre FSx. In this last case, the parallel file system is automatically created and destroyed alongside the HPC cluster via CloudFormation. We chose a `PERSISTENT_2` deployment type, which is best-suited for use cases that have latency-sensitive workloads that require the highest levels of IOPS and throughput, as in the Blue Brain Project's cellular and subcellular simulations. The capacity is set to 1.2TiB (the minimum allowed) with a guaranteed throughput of 250Mbps/TiB, and we enable LZ4 data compression for higher throughput between the object storage servers and targets of Lustre FSx.

The last part of the configuration file refers to the S3 Data Repository Associations created for Lustre FSx. We opted for a read-only S3-DRA to expose the Singularity container images for running jobs in the cluster, and a read/write S3-DRA mount point to expose the knowledge graph assets of Nexus.

### Deployment of a ParallelCluster
With the cluster configuration file defined, deploying the ParallelCluster is relatively easy and straightforward. First, one must ensure that all of the configuration files and scripts retrieved from within the cluster reside on an S3 bucket. The AWS CLI is extremely helpful for this purpose:

```sh
$ aws s3 sync ./config/s3 s3://sboinfrastructureassets/config/
$ aws s3 sync ./scripts/s3 s3://sboinfrastructureassets/scripts/
```

Thereafter, we can simply utilize the ParallelCluster CLI to build the cluster using CloudFormation:

```sh
$ pcluster create-cluster --cluster-configuration ./config/compute-cluster.yaml \
                          --cluster-name sbo-parallelcluster-20241215 \
                          --rollback-on-failure false

$ pcluster describe-cluster --cluster-name sbo-parallelcluster-20241215 \
                            --query clusterStatus
"CREATE_IN_PROGRESS"
...  # Wait inside a loop until the CF stack has finished
$ pcluster describe-cluster --cluster-name sbo-parallelcluster-20241215 \
                            --query clusterStatus
"CREATE_COMPLETE"  # Check for errors using the status
```

And that is all that is required. After the CloudFormation stack has finished, we can connect to the cluster and start scheduling jobs. To do so, we can either query for the head node IP address inside our VPC and then SSH using the master key or any other SSH key installed in the head node:

```sh
$ pcluster describe-cluster --cluster-name sbo-parallelcluster-20241215 \
                            --query headNode.privateIpAddress
10.0.1.2
$ ssh -i ~/.ssh/YOURKEYHERE.pem myuser@10.0.1.2
...
```

Or, alternatively, we can simply use the cluster name and connect via the ParallelCluster CLI using the master SSH key from the cluster configuration file:

```sh
$ pcluster ssh --cluster-name sbo-parallelcluster-20241215 -i aws_coreservices.pem
...
```
