# ParallelCluster Provisioner
> [!NOTE]
> **Also known as:** HPC Resource Provisioner

With the purpose of running large-scale simulations in the Blue Brain Open Platform, the **ParallelCluster Provisioner** is a service that provides the capability for deploying ParalellCluster instances on-demand for a given Project inside a Virtual Lab.

When a new simulation is created, the [Workflow Service](../Workflow/) communicates via REST API with the Provisioner to request the creation of a ParallelCluster that will accommodate the workload from the simulation scheduled in the platform. If required, the Provisioner will then deploy a private HPC cluster that is tailor-made to the necessities of the campaign. Hence, allowing the Workflow Service to manage the different tasks required for the job to succeed.

![ParallelCluster Provisioner - Main Architecture](resources/1_main.drawio.svg)

Here are some of the key technologies utilized for the infrastructure:

- The Provisioner service runs on **AWS Lambda** using a container image stored in **Amazon ECR**.
  - Each container image version is pushed from our private GitLab repository into ECR via a CI/CD pipeline job.
  - The source code of the Provisioner is written in Python 3.
- All the communication with the Provisioner are handled exclusively via a REST API provided by **AWS API Gateway**.
  - Only the Workflow Service is allowed to interact with the provisioner.
- Each new cluster is built and deployed using **AWS ParallelCluster** inside a dedicated **private subnet** in a **private VPC** that features **VPC Endpoints** to access the different services  (see [Network Architecture for ParallelCluster Deployments](#network-architecture-for-parallelcluster-deployments)).
- A cluster deployment consists of a tailor-made HPC cluster that integrates **Amazon EFS** for home directories and essential cluster configuration, **Lustre FSx** as parallel file system, and long-term storage provided by two buckets on **Amazon S3** (see [Storage Architecture for ParallelCluster Deployments](#storage-architecture-for-parallelcluster-deployments)).
  - Only the Workflow Service is allowed to interact with each cluster to schedule jobs via SLURM via SSH[^SSH].
  - Although not implemented yet, it is expected for the Provisoner to generate a unique ED25519 SSH key-pair per deployment via Amazon EC2.
  - It is also expected for the private key of a given ParallelCluster to be stored in AWS Secrets Manager (or AWS Systems Manager Parameter Store).
- Each cluster is configured to communicate with a shared relational database on **Amazon RDS** for SLURM accounting and job monitoring purposes. The account configuration is currently protected via **AWS Secrets Manager**.
  - The RDS deployment is MariaDB-based for compatibility with SLURM.
  - The database instance is deployed via Terraform inside the private VPC and accessible from each subnet.
- Alongside the system log monitoring provided by ParallelCluster by default, each cluster deployment also synchronizes SLURM data to a dedicated **CloudWatch Log Group** containing details about the jobs.
  - The data synchronized includes the SLURM job information, environment (e.g., modules loaded), standard output / error, and other useful information.

[^SSH]: Support for SLURM REST API with JWKS authentication via Keycloak is provided as well.


## Network Architecture for ParallelCluster Deployments
Each ParallelCluster is deployed inside a dedicated subnet of a private VPC with no Internet Gateway. We rely on dedicated **VPC <ins>Gateway</ins> Endpoints** for DynamoDB and S3, as well as **VPC <ins>Interface</ins> Endpoints** for CloudFormation, CloudWatch, EFS, and other services that are required for the whole infrastructure to work accordingly.

![Diagram](resources/2_network.drawio.svg)

A **VPC Peering** connection is set between the Private VPC and the Main VPC to enable certain traffic between both networks. For instance, the Provisioner or even the Workflow Service require communicating with the compute instances deployed. However, we are not interested in these instances accessing the Internet from the Private VPC.

For this purpose, we establish specific **Security Group Rules** to prevent random traffic to cross the boundaries of each VPC. Thus, allowing us to ensure that the ParallelCluster deployments remain completely offline and only accessible by the required services.


## Storage Architecture for ParallelCluster Deployments

Parallel jobs per Virtual Lab Project always run in a private ParallelCluster and are only allowed to `read` / `write` into their own private directory located in the '`scratch`' directory of a private **Lustre FSx** filesystem.

![Diagram](resources/3_storage.drawio.svg)

Scientific data is retrieved through the '`project`' directory in Lustre FSx, which is composed of two S3-DRA mount points targetting two separate S3 buckets for scientific data:

- A '`public`' directory containing all of the public assets of Blue Brain Open Platform stored in a shared S3 bucket. These assets are visible among all of the Virtual Labs.
- A '`private`' directory that contains the assets stored by a particular Project in a Virtual Lab. The assets are implicitly not visible among Projects nor Virtual Labs.

In certain use-cases (e.g., eModel Fitting), Lustre FSx is omitted to optimize the overall operational costs. The simulations rely on EFS instead. The data stored on S3 is still accessible through S3-FUSE, and the data synchronizations are handled through a conventional data upload on the [Metadata and Data Service](../Nexus/).

Note that the data management and knowledge graph data registration is still handled via Nexus, but orchestrated in this case from the Workflow Service.

## Release and Deployment

The provisioner is released as a container image that is published to docker hub at https://hub.docker.com/r/bluebrain/hpc-resource-provisioner. In order to deploy it in our infrastructure as an AWS Lambda it needs to be transferred to an ECR. This ECR, like the lambda, needs to be provisioned through terraform, and so we need a two-phase deployment: one terraform run to set up the ECR, then an action to pull the image from docker and push it to ECR, and then another terraform run to deploy the Lambda.

![Diagram](resources/4_release_and_deploy.drawio.svg)
