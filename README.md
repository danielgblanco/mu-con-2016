# Effortless scalable services in AWS with CloudFormation

Remember when setting up a load-balanced architecture was a tedious process that
involved buying new hardware, configuring networks and routing tables, and
implementing features like SSL termination, asymmetric load or content based
routing? Fortunately, I don't, but thanks to AWS Elastic Load Balancing we can
get the same benefits without any of the hassle.
 
In this short post we will be setting up an AWS Virtual Private Cloud (VPC) with
a auto-scaled set of load-balanced services running the SKIPJAQ public website,
with replicas in each AWS availability zone. In order to do this, we will use
CloudFormation to create and update the stack easily, applying rolling updates
while maintaining high availability.

## Building the stack

The first thing to do is to open a text editor and create a YAML file (e.g. 
`skipjaq-web.yml`). This is where we will define our CloudFormation template,
which we will later tell AWS to create and manage. Our template will contain
two main blocks:

```
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

```
  AmiId:
    Type: String
    Default: ami-e385cb90
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

Here we define a Virtual Private Cloud (VPC) for our stack, isolated from other
resources in the region in its own virtual network, along with a subnet for each
availability zone, each with their respective CIDR blocks:

```
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
```

DNS support is enabled by default for any VPC, with Amazon DNS server resolving
DNS hostnames for instances, but DNS hostnames are not generated automatically
for our instances unless we set the `EnableDnsHostnames` flag.

Notice the use of `!Ref` to reference to other parts of the template. Each AWS
resource has a return type associated with the `Ref` function (ID normally).

#### Access from/to Internet
The VPC created above is completely isolated and has no access to internet.
In order for it to have access to internet we need to create an internet gateway
and route all traffic to internet to it:

```
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
```

With this configuration, instances created in those subnets will not be
routable from outside (automatic public IP mapping is only enabled in default
subnets) but they will be able to initiate connections to outside via the
Internet gateway.

#### Tightening security
Apart from network ACLs, AWS provides a more strict way of limiting network
traffic, Security Groups. The two following security groups will allow traffic
only on port 80 to the ELB and port 8080 to our web servers (coming from the
ELB).

```
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
Now that we have all the networking set up, it is time to add our Elastic Load
Balancer (ELB). The snippet below adds an Internet-facing ELB associated with
our public subnets (one per availability zone).

```
  WebLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Subnets:
      - !Ref WebPublicA
      - !Ref WebPublicB
      SecurityGroups:
      - !Ref WebLBSecurityGroup
  WebTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckIntervalSeconds: 15
      UnhealthyThresholdCount: 8
      HealthCheckPort: 8080
      HealthCheckPath: /health
      Name: skipjaq-web
      Port: 8080
      Protocol: HTTP
      VpcId: !Ref WebVpc
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

We also create a Target Group that will be used to check the status of instances
in our Auto Scaling Group using a specific health check, and a listener on port
80 that will forward requests to our target group.

#### Horizontal scaling
After creating our ELB it is time to add instances that this ELB will direct
traffic to. We first create a Launch Configuration that defines what type
of instance and image we want to use for our replicas, using the values provided
as parameters to the template. Then we create an Auto Scaling Group
that will have a minimum of 4 instances, 2 per availability zone, and an update
policy (more on that later). Lastly, we add a scale-up policy to the Auto
Scaling Group to add one instance if the CPU of an instance goes over 50%
utilisation.

```
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
        - !Ref WebPublicA
        - !Ref WebPublicB
      LaunchConfigurationName: !Ref WebLaunchConfiguration
      MinSize: 4
      MaxSize: 10
      TargetGroupARNs:
        - !Ref WebTargetGroup
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 2
        MaxBatchSize: 2
  WebScaleUpPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      AutoScalingGroupName: !Ref WebAutoScalingGroup
      Cooldown: '15'
      ScalingAdjustment: '1'
  WebHighCpuAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      EvaluationPeriods: '1'
      Statistic: Average
      Threshold: '50'
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

## Creating the stack
Now that we have our CloudFormation template completed, it is time to create it.
To do so, we can use the AWS Console, any of their SDKs or the AWS CLI client.
We will use the latter:

```
aws cloudformation create-stack --stack-name skipjaq-web \
    --template-body \file:///path/to/skipjaq-web.yml \
    --parameters ParameterKey=AmiId,ParameterValue=ami-a123b456,ParameterKey=InstanceType,ParameterValue=m3.medium
```

This assumes that we have credentials and default region configured.

## Performing rolling updates on a running stack

As you saw in the previous points, we added and `UpdatePolicy` to our Auto
Scaling Group to apply updates in batches of 2 and leave always 2 instances
running. Now we will apply a change set to our stack using a CloudFormation
template 