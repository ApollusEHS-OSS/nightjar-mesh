# nightjar-mesh

An AWS ECS Control Mesh with Envoy Proxy

*That's a fancy way to say that Nightjar monitors AWS Elastic Cloud Services for changes, and sends updates to a local Envoy Proxy to change the inbound and outbound traffic routes.*


## About

[Nightjar](https://en.wikipedia.org/wiki/Nightjar) is a **Control Mesh** for Envoy Proxy, designed to run as a Docker sidecar within Amazon Web Services (AWS) Elastic Cloud Services (ECS).

AWS provides their [App Mesh](https://aws.amazon.com/app-mesh/) tooling, but it involves many limitations that some deployments cannot work around, or should not work around.  Nightjar acts as a low-level intermediary between the AWS API and the Envoy Proxy to make deployments in EC2 or Fargate possible, with little fuss.  It even works without `awsvpc` networks, and takes advantage of ephemeral ports!

Nightjar periodically loads the AWS configuration, and sends updates to [Envoy Proxy](https://envoyproxy.github.io/envoy/) to change the host and port for a dynamic list of weighted path mappings.  This works for both inbound traffic into the mesh (a "gateway" service) and for services running inside the mesh ("egress proxy").  The Envoy Proxies themselves manage the network traffic directly to the services, and the services contact the sidecar envoy proxy.


## Some Notes on Terminology

I use the word "mesh" to describe services that can talk to each other as peers.  This is to avoid confusing the term "cluster", which AWS uses to describe the computing resources where ECS tasks run.  Nightjar uses the phrase "namespace", because it splits the different meshes based on AWS Cloud Map namespaces.

## How It Works

**This describes the current Beta version, which is not a control mesh per se, but acts very similar to one.**

You configure the Nightjar container to run inside an [ECS Task Definition](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html), along with a single service container.  The Nightjar container runs the Envoy proxy, and is considered a "sidecar" container here.  The service must be configured to send all traffic to other services in the mesh to the Nightjar container.  Inbound traffic to the service comes from the Nightjar containers running in the other services. 

You configure the Nightjar container with two sets of properties:
    * The local service name.  This tells Nightjar to not handle traffic sent to this local container.  It also indirectly tells Nightjar which mesh it belongs to.
    * The other cluster names.  If you split your network into multiple clusters, then each of the other clusters is defined by name and can direct traffic directly within the other networks.  This is completely optional; if you prefer to have the other clusters have a separate mesh, then you need to have the services connect directly to the other meshs' gateway proxies.

To setup the services, you need to register your tasks using AWS Cloud Map (aka Service Discovery) using SVR registration.  This makes available to Nightjar the service members and how to connect to them.

The beta version is written in Python, and constructs the Envoy Proxy configuration as a static yaml file.  This makes it possible to change the static generation to include extra Envoy features that don't come out of the box.  Eventually, this construction will be moved to a better templating engine to make construction even easier.


## Using Nightjar

Using Nightjar involves registering your services with AWS Cloud Map, adding a specially named service registration instance to cloud map that describes metadata for your service, and adding the Nightjar sidecar container to your services.

Nightjar itself takes these environment variables to configure its operation:

* `SERVICE_MEMBER` (required, no default) the AWS Cloud Map service ID (not the ARN) of the service to which this Nightjar sidecar belongs to.  See the example below for how to set this value.  Set this to `-gateway-` to have Nightjar run in "gateway" mode, where it does not run as a sidecar to another service container, but instead as a gateway proxy into the mesh.
* `SERVICE_PORT` (defaults to 8080) the port number that the envoy proxy will listen for requests that are sent to other services within the `SERVICE_MEMBER` namespace.
* `NAMESPACE_x` where *x* is some number between 0 and 99.  This defines a AWS Cloud Map service namespace other than the `SERVICE_MEMBER` namespace, which Nightjar will forward requests.
* `NAMESPACE_x_PORT` the listening port number that the Envoy proxy will forward requests into the corresponding `NAMESPACE_x` service namespace.  Services send a request to the Nightjar container on this port number to connect to the other namespace.
* `REFRESH_TIME` (defaults to 10) the number of seconds between polling for updates in the configuration. 
* `EXIT_ON_GENERATION_FAILURE` (defaults to 0)  If this value is *anything* other than `0`, then the container will stop if an error occurs while generating the envoy proxy static configuration file.
* `FAILURE_SLEEP` (defaults to 300) if the generation failed, the process will wait this many seconds before stopping the container.  This allows an operator time to inspect the container for problems.
* `ENVOY_LOG_LEVEL` (no default) Sets the Envoy proxy logging level.
* `ENVOY_ADMIN_PORT` (defaults to 9901) the Envoy proxy administration port.
* `AWS_REGION` (required, no default) The AWS region name (i.e. `us-west-2`) in which the Cloud Map registration is managed.
* `DEBUG_CONTAINER` (no default) set this to `1` to start the container as a shell, to help in debugging the container.


## Example of Nightjar with a Service

Control meshes are not easy insert into your network topology.  However, Nightjar attempts to make the configuration as minimal as possible to make the configuration simple and easy to debug.

### AWS Account

First, you need an AWS account with access to a VPC, at least 2 subnets in different activity zones, ECS, an ECS cluster, the ability to create IAM roles, and Cloud Map.  The setup here uses CloudFormation templates to build up the example, but you don't need to use that.  It's just incredibly helpful in keeping the system together.

### CloudFormation Template Setup

The CloudFormation template ("CFT") starts off with some parameters to get us going.  For this example, we will deploy the containers in EC2 instances.  A few tweaks can make this work in Fargate.

```yaml
Parameters:
  ClusterName:
    Type: String
    Description: >
      The ECS cluster name or ARN in which the image registry task will run.

  VPC:
    Type: AWS::EC2::VPC::Id
    Description: >
      The VPC to connect everything to.

  Subnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: >
      The subnets in the VPC for the service to run in.
      Be careful of cross-AZ data charges!

  ECSInstancesSecurityGroup:
    Type: String
    Description: >
      Security group for the EC2 cluster instances.  This is a "shared key" that
      must be given to anything that wants to talk to the EC2 cluster.
```

### Add In Our Service

Let's add in a single service to the stack, called `tuna`.  Note that having a single service means that we don't need a mesh, but this shows all the resources necessary before we add in the mesh.  And it keeps our example simple.

Note that this prepares us for blue/green deployments, by labeling this service's resources with "blue".  To add in a canary test of a service, a copy of the "blue" resources is made with a different color name for only that one service.

```yaml
Resources:
  TunaBlueService:
    Type: "AWS::ECS::Service"
    Properties:
      Cluster: !Ref "ClusterName"
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100
      DesiredCount: 1
      LaunchType: EC2
      TaskDefinition: !Ref "TunaBlueTaskDef"

  TunaBlueTaskDef:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      RequiresCompatibilities:
        - EC2
      # No network definition, which means to use the EC2 standard bridge networking mode.
      ExecutionRoleArn: !Ref TaskExecRole
      TaskRoleArn: !Ref ServiceTaskRole
      Family: "yummy-tuna-blue"
      Tags:
        # Generally good practice to help you out in a production environment.
        - Key: color
          Value: blue
        - Key: service
          Value: tuna
      ContainerDefinitions:
        - Name: service
          Image: !Sub "${AWS:AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/tuna:latest"
          Essential: true
          Cpu: 256
          MemoryReservation: 1024
          PortMappings:
            - ContainerPort: 3000
              # No "HostPort", which makes this an ephemeral port.
              Protocol: "tcp"

  TaskExecRole:
    DependsOn:
    - ServiceTaskRole
    Type: AWS::IAM::Role
    Properties:
      Path: "/"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          -
            Effect: Allow
            Principal:
              Service:
                - "ecs-tasks.amazonaws.com"
            Action:
              - "sts:AssumeRole"
      Policies:
        - PolicyName: DeveloperLocalDockerImagesExec
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - "ecr:GetAuthorizationToken"
                  - "ecr:BatchCheckLayerAvailability"
                  - "ecr:GetDownloadUrlForLayer"
                  - "ecr:BatchGetImage"
                  - "logs:CreateLogStream"
                  - "logs:PutLogEvents"
                Resource:
                  - "*"
              -Effect: Allow
                Action:
                  - "iam:PassRole"
                Resource:
                  - !GetAtt "ServiceTaskRole.Arn"
              - Effect: Allow
                Action:
                  - "ecs:RunTask"
                  - "ecs:StartTask"
                  - "ecs:StopTask"
                Resource:
                  - "*"

  ServiceTaskRole:
    Type: AWS::IAM::Role
    Properties:
      Path: "/"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - "ecs-tasks.amazonaws.com"
          Action:
          - "sts:AssumeRole"
      # Add policies as necessary.
      # This definition here allows for logging
      # and writing to an XRay daemon.
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/CloudWatchFullAccess
        - arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess

      # These policies are for the Nightjar container, which will be
      # added later.  For a real production environment, you will want to
      # separate out the permissions into minimal chunks that allow the
      # container to work.  But, all containers within an ECS service must
      # share the same IAM role.
      Policies:
      - PolicyName: DeveloperLocalDockerImages
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Action:
              - "servicediscovery:List*"
              - "servicediscovery:Get*"
              Resource:
              - "*"
```

This creates the service, which starts the container in our ECS cluster.  The only way to connect to this service is through the ephemeral port on the EC2 instance, which is above the 32768 range.

On a side note, using ephemeral ports means that you must have a large number of ports open in a security group to allow access to any of the containers.  The alternative approach is to explicitly restrict container ports to a specific host port, but that makes the configuration less flexible.  Either way you do this, Nightjar will detect the correct port.

### Add in Service Discovery

Nightjar uses AWS Cloud Map for storing the configuration information and location of all the services and their container instances.  Below is the additional and changed resources to the growing CFT.  While we're here, we'll name the mesh `yummy`, because tuna can be yummy (YMMV). 

```yaml
Resources:
  # The namespace for this mesh.  If we have multiple meshes, each one has its
  # own namespace.  The primary reason to split separate clusters is due to
  # overlapping listening paths for the services.  Another reason is better network
  # traffic control.  It can also add extra security by limiting which paths
  # an outside service can talk to by use of the "private" paths. 
  YummyNamespace:
    Type: "AWS::ServiceDiscovery::PrivateDnsNamespace"
    Properties:
      Description: "The Yummy Mesh"
      Name: "yummy.service.local"
      Vpc: !Ref VPC

  # All the different color deployments of the Tuna have their
  # own discovery record.  
  TunaBlueServiceDiscoveryRecord:
    Type: 'AWS::ServiceDiscovery::Service'
    Properties:
      Name: tuna-blue
      DnsConfig:
        NamespaceId: !Ref "YummyNamespace"
        DnsRecords:
          # Containers add themselves into this record, and with the SRV
          # type, they register the IP and the ephemeral port they listen on. 
          - Type: SRV
            TTL: 300
      HealthCheckCustomConfig:
        FailureThreshold: 1

  # A data-only service discovery instance.  Each instance includes one of these to
  # tell Nightjar additional meta-data about the specific service.  This includes
  # the different paths.
  TunaBlueReferenceInstance:
    Type: AWS::ServiceDiscovery::Instance
    Properties:
      # The instance ID MUST be "service-settings"; Nightjar looks for this ID.
      InstanceId: service-settings
      ServiceId: !Ref "TunaBlueServiceDiscoveryRecord"
      InstanceAttributes:
        # High level information about the service/color.
        SERVICE: tuna
        COLOR: blue

        # If your service uses HTTP2, then set this attribute and value.
        HTTP2: enabled
        
        # List of all the URI path prefixes that receive traffic.
        # The relative weight to assign this service/color for this path.
        "/tuna": "100"
        
        # These settings are required for SRV records, but for this
        # record, the values are never used.  So we set these to valid
        # values that are harmless.
        AWS_INSTANCE_IPV4: 127.0.0.1
        AWS_INSTANCE_PORT: 1234

  # Update the service to include registration 
  TunaBlueService:
    Type: "AWS::ECS::Service"
    Properties:
      Cluster: !Ref "ClusterName"
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100
      DesiredCount: 1
      LaunchType: EC2
      TaskDefinition: !Ref "TunaBlueTaskDef"
      ServiceRegistries:
        - RegistryArn: !GetAtt "TunaBlueServiceDiscoveryRecord.Arn"
          # The container name and port of the service we're registering.
          ContainerName: service
          ContainerPort: 3000
```

If the Tuna container goes down, or if it scales up with 16 running containers, then the service discovery instances are also updated to reflect that changing topology.  That's part of the magic of that `ServiceRegistries` key.

That seems like a lot of work to setup just one container, and it is, but that's the boilerplate we need to get started with a mesh.

### Add In Nightjar

With all that boilerplate out of the way, adding in Nightjar to the template is now trivial.  It's just adding in a new container to the existing task definition with some special properties, and adding a link from the service to the Nightjar container.

```yaml
  TunaBlueTaskDef:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      RequiresCompatibilities:
        - EC2
      # No network definition, which means to use the EC2 standard bridge networking mode.
      ExecutionRoleArn: !Ref TaskExecRole
      TaskRoleArn: !Ref ServiceTaskRole
      Family: "yummy-tuna-blue"
      Tags:
        # Generally good practice to help you out in a production environment.
        - Key: color
          Value: blue
        - Key: service
          Value: tuna
      ContainerDefinitions:
        - Name: service
          Image: !Sub "${AWS:AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/tuna:latest"
          Essential: true
          Cpu: 256
          MemoryReservation: 1024
          PortMappings:
            - ContainerPort: 3000
              # No "HostPort", which makes this an ephemeral port.
              Protocol: "tcp"
          
          # New!
          Link:
            - nightjar

        # New container!
        - Name: nightjar
          # Note: this is not a real image name.  You need to build it yourself.
          Image: locally/built/nightjar
          User: "1337"
          Essential: true
          Memory: 128
          Ulimits:
          - Name: nofile
            HardLimit: 15000
            SoftLimit: 15000
          Environment:
          # These environment variables must be carefully set.

          # The AWS region, so Nightjar can ask for the right resources.
          - Name: AWS_REGION
            Value: !Ref "AWS::Region"

          # The nightjar-running Envoy proxy's logging level.
          - Name: ENVOY_LOG_LEVEL
            Value: info

          # The Envoy proxy administration port.  Defaults to 9901
          - Name: ENVOY_ADMIN_PORT
            Value: 9901
    
          # Which service record that defines the service in which Nightjar runs.
          - Name: SERVICE_MEMBER
            Value: !Ref "TunaBlueServiceDiscoveryRecord"

          # The port number that the envoy proxy will listen to for connections
          # *from* the sidecar service *to* the mesh.
          - Name: SERVICE_PORT
            Value: 8090
          PortMappings:
          - ContainerPort: 9901
            Protocol: tcp
          - ContainerPort: 8090
            Protocol: tcp
          HealthCheck:
            Command:
            - "CMD-SHELL"
            - "curl -s http://localhost:9901/server_info | grep state | grep -q LIVE"
            Interval: 5
            Timeout: 2
            Retries: 3
          
```

In this example, the Nightjar Envoy Proxy will listen to port 8090 for connections from the Tuna service to the the rest of the mesh.  Because the Nightjar container is named `nightjar`, and the Tuna service includes a link to `nightjar`, the Tuna service should call to `http://nightjar:8090` plus the other service's path to connect to it.


### Adding Another Service

If we want to add another service to the mesh, it's mostly cut-n-paste of the above.  The one thing to note is that *the tuna service setup does not change.*  Nightjar picks up the new service from the namespace and adds in the weighted paths.  This can be done even without stopping the Tuna service.


### Adding a Mesh Gateway

You could use a standard Application Load Balancer for every service, but that means you don't gain the great network shaping that Envoy gives us.  Instead, we want to take advantage of the envoy goodness, but that means introducing a gateway service that all outside traffic uses to access the mesh, and that includes a load balancer.

Nightjar produces this with just a few tweaks.  You'll want to set the nightjar container as the only container in the gateway, and configure it as a gateway.

```yaml
  # Setup the gateway service + container definition.
  GatewayService:
    Type: AWS::ECS::Service
    DependsOn:
      - GatewayTargetGroup
      - GatewayLoadBalancerListener
    Properties:
      Cluster: !Ref "ClusterName"
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100
      DesiredCount: 1
      LaunchType: EC2
      # The gateway is accessible through a load balancer.
      LoadBalancers:
        - ContainerName: nightjar
          ContainerPort: 2000
          TargetGroupArn: !Ref GatewayTargetGroup
      TaskDefinition: !Ref GatewayTaskDef
  
  GatewayTaskDef:
    Type: "AWS::ECS::TaskDefinition"
    DependsOn:
      - YummyNamespace
    Properties:
      RequiresCompatibilities:
      - EC2
      ExecutionRoleArn: !Ref TaskExecRole
      TaskRoleArn: !Ref ServiceTaskRole
      Family: "yummy-gateway"
      Tags:
        - Key: color
          Value: gateway
        - Key: service
          Value: gateway
      ContainerDefinitions:
        - Name: nightjar
          Image: locally/built/nightjar
          User: "1337"
          Essential: true
          Memory: 128
          Ulimits:
          - Name: nofile
            HardLimit: 15000
            SoftLimit: 15000
          Environment:
          # To configure Nightjar as a gateway, we assign it a special reserved name.
          - Name: SERVICE_MEMBER
            Value: "-gateway-"
          # It will not think of itself as part of a service, so it won't ignore any
          # paths.  Instead, it listens for connections on the mesh, defined using
          # the namespace information.
          - Name: NAMESPACE_1
            Value: !Ref YummyNamespace
          - Name: NAMESPACE_1_PORT
            # This is the port that the load balancer is forwarding to.
            Value: "2000"

          # All these are the same...
          - Name: AWS_REGION
            Value: !Ref "AWS::Region"
          - Name: ENVOY_LOG_LEVEL
            Value: info
          - Name: ENVOY_ADMIN_PORT
            Value: 9901
          PortMappings:
          - ContainerPort: 9901
            Protocol: tcp
          - ContainerPort: 2000
            Protocol: tcp
          HealthCheck:
            Command:
            - "CMD-SHELL"
            - "curl -s http://localhost:9901/server_info | grep state | grep -q LIVE"
            Interval: 5
            Timeout: 2
            Retries: 3

  GatewayTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: gateway-tg
      # The listening port of the container, which nightjar will forward to the mesh
      Port: 2000
      Protocol: HTTP
      VpcId: !Ref VPC
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: 120
      HealthCheckIntervalSeconds: 60
      # Hard-coded health check path on the envoy proxy.
      HealthCheckPath: '/ping'
      Matcher:
        HttpCode: "200"
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2

  GatewayLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internal
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value: '300'
      Subnets: !Ref Subnets
      SecurityGroups:
        - !Ref LoadBalancerSecurityGroup
        - !Ref ECSInstancesSecurityGroup
  GatewayLoadBalancerListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    DependsOn:
      - GatewayLoadBalancer
    Properties:
      DefaultActions:
        - TargetGroupArn: !Ref GatewayTargetGroup
          Type: 'forward'
      LoadBalancerArn: !Ref 'GatewayLoadBalancer'
      # The load balancer listening port.
      Port: 80
      Protocol: HTTP

  LoadBalancerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Access to the fronting load balancer
      VpcId: !Ref VPC
      SecurityGroupIngress:
          # You should probably change this to something safer.
          - CidrIp: 0.0.0.0/0
            IpProtocol: -1

```

If you have paths you want Nightjar to not make available in the gateways (they are "private" within the mesh), then, in the `service-settings` service registration instance, prefix the path with a question mark:

```yaml
  TunaBlueReferenceInstance:
    Type: AWS::ServiceDiscovery::Instance
    Properties:
      InstanceId: service-settings
      ServiceId: !Ref "TunaBlueServiceDiscoveryRecord"
      InstanceAttributes:
        AWS_INSTANCE_IPV4: 127.0.0.1
        AWS_INSTANCE_PORT: 1234
        SERVICE: tuna
        COLOR: blue
        # public path; the gateway forwards these.
        "/tuna": "65"
        
        # private path; the gateway does not forward these.
        "?/albacore": "100"
```
