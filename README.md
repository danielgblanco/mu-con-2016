# Effortless scalable services in AWS with CloudFormation (Part 1)

Remember when setting up a scalable load-balanced architecture was a tedious
process that involved buying new hardware, configuring networks and routing
tables, and implementing features like SSL termination, asymmetric load or
content based routing? Fortunately, I don't, but thanks to AWS Elastic Load
Balancing  and Auto Scaling Groups we can get the same benefits without any of
the hassle.
 
In this short post we will be setting up an AWS Virtual Private Cloud (VPC)
[\[1\]](#link1) with an auto-scaled set of load-balanced services running the SKIPJAQ
public website, using Elastic Load Balancing (ELB) [\[2\]](#link2) and Auto Scaling 
Groups (ASG) [\[3\]](#link3). In order to do this, we will use CloudFormation
[\[4\]](#link4) to create and update the stack easily, applying rolling updates while
maintaining high availability.

The following diagram illustrates the stack we will be creating;

![Architecture](architecture.png)

## Building the stack

The first thing to do is to open a text editor and create a YAML file (e.g. 
`skipjaq-web.yml`). This is where we will define our CloudFormation template,
which we will later tell AWS to create and manage. Our template will contain
two main blocks:

```yaml
Parameters:
  # Template parameters
Resources:
  # AWS resources
```

### Template parameters

CloudFormation allows users to pass parameters when applying a template. These
can be referenced later in the template to achieve better re-usability. We
will define an `AmiId` and an `InstanceType` that will use to create our
website replicas:

```yaml
  AmiId:
    Type: String
    Default: ami-93cf9ae0
  InstanceType:
    Type: String
    Default: t2.medium
```

Parameters can have default values that will be picked up in case no value
is passed when the template is executed.

### AWS resources

The following are the AWS resources that will be created with this template.
Each of the operations in CloudFormation is transactional, so if one of the
resources fails to create it will rollback deleting everything that was created,
or going back to a previous version in case of updates.

#### Basic networking

Here we define a VPC for our stack, isolated from other resources in the region
in its own virtual network, along with a private and  a public subnet for each
availability zone, each with their respective CIDR blocks [\[5\]](#link5):

```yaml
  WebVpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: skipjaq-web
  WebPublicA:
      Type: AWS::EC2::Subnet
      Properties:
        CidrBlock: 10.0.1.0/24
        AvailabilityZone: eu-west-1a
        VpcId: !Ref WebVpc
  WebPublicB:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: eu-west-1b
      VpcId: !Ref WebVpc
  WebPublicC:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.3.0/24
      AvailabilityZone: eu-west-1c
      VpcId: !Ref WebVpc
  WebPrivateA:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.100.0/24
      AvailabilityZone: eu-west-1a
      VpcId: !Ref WebVpc
  WebPrivateB:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.101.0/24
      AvailabilityZone: eu-west-1b
      VpcId: !Ref WebVpc
  WebPrivateC:
    Type: AWS::EC2::Subnet
    Properties:
      CidrBlock: 10.0.102.0/24
      AvailabilityZone: eu-west-1c
      VpcId: !Ref WebVpc
```

The reason to create private and public subnets is because our instances will
be hosted in private networks without any incoming or outgoing access to
Internet. All traffic will go via the ELB, that will be associated with the
public subnets (more on that later).

DNS support is enabled by default for any VPC, with Amazon DNS server resolving
DNS hostnames for instances, but DNS hostnames are not generated automatically
for our instances unless we set the `EnableDnsHostnames` flag.

Notice the use of `!Ref` to reference to other parts of the template. Each AWS
resource has a return type associated with the `Ref` function (ID normally).

#### Access from/to Internet
The VPC created above is completely isolated and has no access to internet.
In order for it to have access to internet we need to create an internet gateway
and route all traffic to internet to it [\[6\]](#link6):

```yaml
  WebInternetGateway:
    Type: AWS::EC2::InternetGateway
  WebInternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref WebInternetGateway
      VpcId: !Ref WebVpc
  WebPublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref WebVpc
  WebPublicRouteToInternet:
    Type: AWS::EC2::Route
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref WebInternetGateway
      RouteTableId: !Ref WebPublicRouteTable
  WebPublicRouteTableSubnetA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref WebPublicRouteTable
      SubnetId: !Ref WebPublicA
  WebPublicRouteTableSubnetB:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref WebPublicRouteTable
      SubnetId: !Ref WebPublicB
  WebPublicRouteTableSubnetC:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref WebPublicRouteTable
      SubnetId: !Ref WebPublicC
```

As previously mentioned, only our public networks will have Internet access,
with our instances being protected in private networks not reachable from
outside.

#### Tightening security
Apart from network ACLs, AWS provides a more strict way of limiting network
traffic, Security Groups. The two following security groups will allow traffic
only on port 80 to the ELB and port 8080 to our web servers (coming only from
the ELB).

```yaml
  WebLBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for web load balancer
      VpcId: !Ref WebVpc
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
  WebTargetsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for web targets
      VpcId: !Ref WebVpc
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 8080
        ToPort: 8080
        SourceSecurityGroupId: !Ref WebLBSecurityGroup
```

It is worth noting that AWS allows all outgoing traffic if no egress rules are
given.

#### Adding the load balancer
Now that we have all the networking set up, it is time to add our ELB. The
snippet below adds an Internet-facing ELB associated with our public subnets,
one per availability zone. You might be wondering why we create 3 public
subnets when they're not really going to host any resources (you won't see
any EC2 instances created there as part of the ELB). This is because an
Internet-facing ELB needs to be associated with at least 2 public subnets, in
different availability zones within your VPC.

```yaml
  WebLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Subnets:
      - !Ref WebPublicA
      - !Ref WebPublicB
      - !Ref WebPublicC
      SecurityGroups:
      - !Ref WebLBSecurityGroup
```
   
The ELB needs some targets to forward traffic to. Here we define where those
targets can be reached (port 80 HTTP) and the health checks to make sure that a
web server is ready to accept requests. This will especially important when
applying updates to make sure that there is always a number of replicas ready
to accept requests.
   
```yaml
  WebTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 8080
      Protocol: HTTP
      HealthCheckIntervalSeconds: 15
      UnhealthyThresholdCount: 8
      HealthCheckPort: 8080
      HealthCheckPath: /health
      VpcId: !Ref WebVpc
```

Now that we have an ELB and a Target Group we can add the configuration to have
the ELB listen on port 80 for HTTP and forward all requests to the target group
configured above.

```yaml
  WebServiceListener:
    Type : AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref WebTargetGroup
      LoadBalancerArn: !Ref WebLoadBalancer
      Port: 80
      Protocol: HTTP
```

#### Horizontal scaling
After creating our ELB it is time to add instances that this ELB will direct
traffic to as part of a target group. The first step is to create a Launch
Configuration that defines what type of instance and image we want to use for
our instances, using the values provided as parameters to the template.

With this Launch Configuration we can create an Auto Scaling Group that will
have a minimum of 6 instances, 2 per availability zone, and maximum of 15,
associated with our private networks.

```yaml
  WebLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: !Ref AmiId
      InstanceType: !Ref InstanceType
      SecurityGroups:
        - !Ref WebTargetsSecurityGroup
  WebAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - !Ref WebPrivateA
        - !Ref WebPrivateB
        - !Ref WebPrivateC
      LaunchConfigurationName: !Ref WebLaunchConfiguration
      MinSize: 6
      MaxSize: 15
      TargetGroupARNs:
        - !Ref WebTargetGroup
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 3
        MaxBatchSize: 3
```

Notice that our instances cannot initiate a connection to Internet and they
also do not have SSH access or a key associated with them, we will cover these
topics in part two of this series.

The `UpdatePolicy` [\[7\]](#link7) attribute specifies how the ASG handles updates.
We will talk more about that later but for now we can say that it will apply
updates in batches of 3 always leaving at least 3 instances accepting requests.

The default behaviour of an ASG is to create the minimum number of instances
and use our defined health checks (or the instance status if none given) to
maintain that number. The number of instances in the ASG would need to be
changed manually. To react quickly to periods of heavy load, we can enable
Dynamic Scaling [\[8\]](#link8) to _scale out_ according to certain metrics. In the
example below, we define a scaling policy that will add one instance to the ASG
and then wait for a cool-down period before adding more instances. This scaling
event is triggered by an alarm that will be raised when the average CPU usage of
one instance is higher than 60% for a minute.

```yaml
  WebScaleUpPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      AutoScalingGroupName: !Ref WebAutoScalingGroup
      Cooldown: '120'
      ScalingAdjustment: '1'
  WebHighCpuAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      EvaluationPeriods: '1'
      Statistic: Average
      Threshold: '60'
      AlarmDescription: CPU too high or metric does not exist (instance down)
      Period: '60'
      AlarmActions:
      - !Ref WebScaleUpPolicy
      Namespace: AWS/EC2
      Dimensions:
      - Name: AutoScalingGroupName
        Value: !Ref WebAutoScalingGroup
      ComparisonOperator: GreaterThanThreshold
      MetricName: CPUUtilization
```

We don't have a _scale in_ policy defined. This means that removing instances
from the ASG will have to be done manually. The way that instances are
terminated can also be configured, but if not defined the default
policy is designed to help ensure that our network architecture spans
Availability Zones evenly [\[9\]](#link9).

## Creating the stack
Now that we have our CloudFormation template completed, it is time to create it.
To do so, we can use the AWS Console, any of their SDKs or the AWS CLI client.
We will use the latter:

```bash
aws cloudformation create-stack --stack-name skipjaq-web \
    --template-body file:///path/to/skipjaq-web.yml \
    --parameters ParameterKey=AmiId,ParameterValue=ami-a123b456,ParameterKey=InstanceType,ParameterValue=m3.medium
```

This assumes that we have credentials and default region configured. We can
check the status of the resources in the stack with the following command:

```bash
aws cloudformation describe-stack-resources --stack-name skipjaq-web
```

## Performing rolling updates on a running stack

As you saw in the previous points, we added and `UpdatePolicy` to our Auto
Scaling Group to apply updates in batches of 3 and leave always 3 instances
running. To apply an update to our stack we have two options: we can either
update the whole stack with a new template (AWS will check what needs to be 
changed and apply the necessary changes) or we can use Change Sets 
[\[10\]](#link10). The advantage of using Change Sets is that it is possible to
review what changes are going to be made before applying them, and even have
different levels of authorisation to create or execute Change Sets. We will now 
apply a change set to our stack using a CloudFormation Change Sets. To do this,
we can use the same template we already have, but this time we change the
`AmiId` parameter to update the Launch Configuration. We start by creating a
Change Set:

```bash
aws cloudformation create-change-set --stack-name skipjaq-web \
    --change-set-name skipjaq-web-new-image
    --use-previous-template
    --parameters ParameterKey=AmiId,ParameterValue=ami-c987d654,ParameterKey=InstanceType,ParameterValue=m3.medium
```

We review the changes that will be made before proceeding:

```bash
aws cloudformation describe-change-set --change-set-name skipjaq-web-new-image
```

And then we execute the change set:

```bash
aws cloudformation execute-change-set --change-set-name skipjaq-web-new-image
```

The rolling update will be applied in two batches of 3. It will create 3 new
instances, one per availability zone, and wait for them to be available. Then,
once the instances are ready to accept requests, it will delete one instance
from each availability zone according to the termination policy. This will
then continue until all instances are replaced [\[11\]](#link11).

Using rolling updates make our stack remain available at all times, with
instances serving requests in all availability zones in a region.

## External links

<a id="link1"/>\[1\]: [Virtual Private Cloud][1] - General page for AWS Virtual
Private Networks

<a id="link2"/>\[2\]: [Elastic Load Balancing][2] - General page for AWS Elastic
Load Balancing

<a id="link3"/>\[3\]: [Auto Scaling][3] - General page for AWS Auto Scaling

<a id="link4"/>\[4\]: [AWS CloudFormation][4] - General page for AWS
CloudFormation

<a id="link5"/>\[5\]: [Your VPC and Subnets][5] - VPC and subnet routing and
best practices

<a id="link6"/>\[6\]: [Internet Gateways][6] - How to enable Internet access in
a VPC

<a id="link7"/>\[7\]: [Update Policy][7] - How to define Update Policies for an
ASG with all the possible options

<a id="link8"/>\[8\]: [Dynamic Scaling][8] - What Dynamic Scaling is and how it
is configured

<a id="link9"/>\[9\]: [Instance Termination][9] - Different types of instance
termination, including how AWS terminates them by default

<a id="link10"/>\[10\]: [CloudFormation Change Sets][10] - Information about
CloudFormation Change Sets and how they are applied

<a id="link11"/>\[11\]: [Rolling Updates with CloudFormation][11] - Short
article explaining how rolling updates work with CloudFormation

[1]: https://aws.amazon.com/vpc "Virtual Private Cloud"
[2]: https://aws.amazon.com/elasticloadbalancing "Elastic Load Balancing"
[3]: https://aws.amazon.com/autoscaling "Auto Scaling"
[4]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html "AWS CloudFormation"
[5]: http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html "Your VPC and Subnets"
[6]: http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Internet_Gateway.html "Internet Gateways"
[7]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html "Update policy"
[8]: http://docs.aws.amazon.com/autoscaling/latest/userguide/as-scale-based-on-demand.html "Dynamic Scaling"
[9]: http://docs.aws.amazon.com/autoscaling/latest/userguide/as-instance-termination.html "Instance Termination"
[10]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html "CloudFormation Change Sets"
[11]: https://cloudonaut.io/rolling-update-with-aws-cloudformation "Rolling updates"