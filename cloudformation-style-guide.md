# Cloud303 CloudFormation Style Guide - draft v0.0.1

<img alt="cloudformation" src="readme-images/cloudformation-logo.png" width="300px">

<img alt="cloud303" src="readme-images/cloud303-logo.png" width="300px">

There's more than 1 way to write a CloudFormation template, but this document sets forth  some best practice guidelines.

## Table of Contents

- [Cloud303 CloudFormation Style Guide - draft v0.0.1](#cloud303-cloudformation-style-guide---draft-v001)
  - [Table of Contents](#table-of-contents)
  - [Language](#language)
  - [Comments](#comments)
  - [Intrinsic Functions](#intrinsic-functions)
  - [Conditions](#conditions)
  - [Parameters](#parameters)
  - [Metadata](#metadata)
  - [Resources](#resources)
  - [Outputs](#outputs)
  - [Stack Architecture](#stack-architecture)
  - [Attribution](#attribution)

## Language

  <a name="language--yaml"></a><a name="1.1"></a>
  - [1.1](#language--yaml) **YAML**: Use YAML for authoring templates. It is possible to write templates in JSON. Do not write templates in JSON.

    > Why?

    + YAML is far more human-readable than JSON, even without commenting. However...
    + YAML allows for comments. JSON does not. Obviously, comments can be helpful for documenting work, but they are also useful for commenting out entire resources while testing. 
    + Who would voluntarily want to deal with all those brackets?


  <a name="language--quotes"></a><a name="1.2"></a>
  - [1.2](#language--quotes) Unless absolutely necessary (as below) do not wrap your strings in quotes. It is not necessary. 

    ```yaml
    # bad
    wafSqlInjRule01:
        Type: AWS::WAFRegional::Rule
        Properties:
        MetricName: "BlockSQLInjections"
    
    # bad
    wafSqlInjRule01:
        Type: AWS::WAFRegional::Rule
        Properties:
        MetricName: 'BlockSQLInjections'
    
    # good
    wafSqlInjRule01:
        Type: AWS::WAFRegional::Rule
        Properties:
        MetricName: BlockSQLInjections

    ```

  <a name="language--quotes-must"></a><a name="1.3"></a>
  - [1.3](#language--quotes-must) If you must use quotes, such as in a policy document, use `"double quotes"`.
  
    ```yaml
    # bad
    AccountId: '012345678910'

    # good
    AccountId: "012345678910"
    ```
   <a name="language--numbers"></a><a name="1.4"></a>
  - [1.4](#language--numbers) Another place where you might need quotes is to surround numbers starting with a zero. YAML treats numbers that start with 0 as octal. 

    ```yaml
    # bad
    AccountId: 012345678910

    # good
    AccountId: "012345678910"
    ```

## Comments

  <a name="comments--singleline"></a><a name="2.1"></a>
  - [2.1](#comments--singleline) Place single line comments on a newline above the subject. Put an empty line before the comment unless it is the first line of a resource block or property block.

    ```yaml
    # bad
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        AccessControl: PublicRead
        BucketName: !Ref WebappSourceBucketName #Bucket needs to be explicitly named due to ...

    # good
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        AccessControl: PublicRead

        # Bucket needs to be explicitly named due to ...
        BucketName: !Ref WebappSourceBucketName

    # also good
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        # Bucket needs to be explicitly named due to ...
        BucketName: !Ref WebappSourceBucketName
    ```

  <a name="comments--space"></a><a name="2.3"></a>
  - [2.2](#comments--space) Start all comments with a space after #. This makes it easier to read.

    ```yaml
    #bad
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        #Bucket needs to be explicitly named due to ...
        BucketName: !Ref WebappSourceBucketName

    # good
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        # Bucket needs to be explicitly named due to ...
        BucketName: !Ref WebappSourceBucketName
    ```

## Intrinsic Functions

  <a name="functions--prefer-shorthand"></a><a name="3.1"></a>
  - [3.1](#functions--prefer-shorthand) Use shorthand for all intrinsic functions, unless this is not possible. See [below] for nesting intrinsic functions.

    > Why? Shorthand is easier to read and less error prone.

    ```yaml
    # bad
    Ref: InstanceType

    # good
    !Ref InstanceType
    
    # bad
    Name:
        Fn::Sub:
            appName-${AWS::Region}

    # good
    Name: !Sub appName-${AWS::Region}
    
    # bad
    - Fn::FindInMap:
        - AWSInstanceType2Arch
        - Ref: InstanceType
        - Arch

    # good
    !FindInMap [AWSInstanceType2Arch, !Ref InstanceType, Arch]
    ```
<a name="functions--no-join"></a><a name="3.1"></a>
  - [3.1](#functions--no-join) Use !Sub (Fn::Sub) instead of !Join (Fn::Join) wherever possible. !Sub is cleaner, easier to read and allows for in-line variable substitution (which we do a LOT). 

```yaml
# bad
UserData:
  !Base64:
    Fn::Join:
    - ''
    - - "#!/bin/bash -xe\n"
      - "yum update -y aws-cfn-bootstrap\n"
      - "/opt/aws/bin/cfn-init -v "
      - "         --stack "
      - Ref: AWS::StackName
      - "         --resource EC2Instance "
      - "         --region "
      - Ref: AWS::Region
      - "\n"
      - "/opt/aws/bin/cfn-signal -e $? "
      - "         --stack "
      - Ref: AWS::StackName
      - "         --resource EC2Instance "
      - "         --region "
      - Ref: AWS::Region
      - "\n"

    # good
    UserData:
    Fn::Base64:
        !Sub |
        #!/bin/bash -xe
        yum update -y aws-cfn-bootstrap
        /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}
        /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}

    # bad
    NotificationARN: { "Fn::Join": [ ":", [ "arn:aws:sns", { "Ref": "AWS::Region" },{ "Ref": "AWS::AccountId" }, "SnsTopicName" ] ] }

    # good
    NotificationARN: !Sub arn:aws:sns:${AWS::Region}:${AWS::AccountId}:SnsTopicName


```

## Conditions

  <a name="conditions--avoid-complex"></a><a name="4.1"></a>
  - [4.1](#conditions--avoid-complex) Naming

    > Conditions should be camel-case as well, but they should begin with "Cond" (with an upper-case "c").

    ```yaml
    # bad 
    condInternalAlb: !Equals [internal, !Ref pAlbType]
    
    # bad 
    InternalAlbCondition: !Equals [internal, !Ref pAlbType]
    
    # good 
    CondInternalAlb: !Equals [internal, !Ref pAlbType]

    
    ```


## Parameters

  <a name="parameters--names"></a><a name="5.1"></a>
  - [5.1](#parameters--names) Parameters should be camel-case and begin with a lower-case "p". This is to make it impossible to confuse a parameter with a resource.     

    ```yaml
    # bad
    albType:
        Type: String
        Description: Internal or Internet-Facing ALB
        Default: internet-facing
        AllowedValues:
            - internal
            - internet-facing

    # good
    pAlbType:
        Type: String
        Description: Internal or Internet-Facing ALB
        Default: internet-facing
        AllowedValues:
            - internal
            - internet-facing

    # bad
    enableWaf:
        Type: String
        Description: Create and integrate Web Application Firewall with the ALB
        Default: false
        AllowedValues:
            - true
            - false

    # good
    pEnableWaf:
        Type: String
        Description: Create and integrate Web Application Firewall with the ALB
        Default: false
        AllowedValues:
            - true
            - false
    ```

## Metadata
  <a name="metadata--blocks-explained"></a><a name="6.1"></a>
- [6.1](#metadata--blocks-explained) Always use metadata to make the parameters as understandable to the layman as possible. 

    * While not necessary for a template to deploy, metadata is a crucial to allowing clients (and even engineers who didn't write the template!) to understand our templates. Here's how we do that:

    * The "Parameters" page in the AWS console is created using data from 2 blocks in the template: 

        * The `Parameters` block 
            - This is where a parameter's internal name, type, description, default value and allowed values are defined. 
        * The `Metadata` block (2 sections)
            - The `ParameterLabels` section
                - This is where the "friendly" name of the parameter is defined.
            - The `ParameterGroups` section
                - This is where the parameters are organized for the sake of the UI.  

        * Below, we will define a parameter for choosing an EC2 instance type (called `pEc2InstanceType`) and explain how to label and organize it:

```yaml

# Parameters Section: Internal definition for parameters named
# pEc2InstanceName, pEc2InstanceAMI, & pEc2InstanceType.
   
   pEc2InstanceName:
    Type: String
    Description: EC2 Instance Name
  
  pEc2InstanceAMI:
    Type: String 
    Description: Default AMI ID is based in us-east-1. Ensure Ubuntu AMI is specific to the region. 
    Default: ami-085925f297f89fce1 

  pEc2InstanceType:
    Type: String
    Description: Instance Family and size
    Default: t3.small
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
      - t3.large
      - m4.large
      - m4.xlarge
      - m4.2xlarge
      - m4.4xlarge
      - r4.large
      - r4.xlarge
      - r4.2xlarge
      - r4.4xlarge
      - m5.large
      - m5.xlarge
      - m5.2xlarge
      - m5.4xlarge

# Beginning of the Metadata section (required)

Metadata:
  AWS::CloudFormation::Interface:

# Metadata -> ParameterLabels Section - 
# Creates a label (friendly name to replace "pEc2InstanceType")

      ParameterLabels:
          pEc2InstanceName:
            default: EC2 Instance Name
          pEc2InstanceAMI:
            default: EC2 Instance AMI
          pEc2InstanceType:
            default: EC2 Instance Type
          

# Metadata -> ParameterGroups Section -
# Creates a section on the Parameters page 
# with the label "EC2 Instance Settings" which groups 
# our parameters "pEc2InstanceName, pEc2InstanceAMI, & pEc2InstanceType" 
# with other parameters that affect the settings of the EC2 instance 

        ParameterGroups:
          - Label:
              default: EC2 Instance Settings
            Parameters:
              - pEc2InstanceName
              - pEc2InstanceAMI
              - pEc2InstanceType
```
* The end result of all this work is an easily readable `Parameters` section:

<img alt="console-screenshot" src="readme-images/console-screenshot.png" width="900px">


## Resources

  <a name="resources--logical-resource-id"></a><a name="7.1"></a>
  - [7.1](#resources--logical-resource-id) Resource names should be camel-case and begin with a lower-case letter. 

    ```yaml
    # bad
    WebAppLogBucket:
      Type: AWS::S3::Bucket
      Properties:
        ...

    # good
    webAppLogBucket:
      Type: AWS::S3::Bucket
      Properties:
        ...
    ```

  <a name="resources--physical-resource-id"></a><a name="7.2"></a>
  - [7.2](#resources--physcial-resource-id) Use simple, descriptive logical resource names that make it clear what the resource is. 

    ```yaml
    # bad
    appLogs:
      Type: AWS::S3::Bucket
      Properties:
        ...

    # good
    webAppLogBucket:
      Type: AWS::S3::Bucket
      Properties:
        ...
    ```

<a name="resources-add-account-number"></a><a name="7.3"></a>
  - [7.3](#resources--add-account-number) When naming a resource, when in doubt, add the region and account number to the end of the name using the `!Sub` function. This is especially critical with S3 buckets, as their names are global and making them unique by guessing is a waste of time. 

  ```yaml
    # bad
    Name: my-awesome-cloudtrail-bucket
    # good
    Name: cloudtrail-${AWS::Region}-${AWS::AccountId}
  ```

## Outputs

  <a name="outputs--export-names"></a><a name="8.1"></a>
  - [8.1](#outputs--export-names) Be sure to include a "templateVersion" key in your Outputs section so we can keep track of which version is deployed in which clients' accounts. 

    ```yaml
    # bad
    Outputs: 

    # goodOutputs: 
    Outputs:  
      Version:
        Description: Template Version
        Value: cis-benchmark-monitoring-suite-3.0b1
    ```
  <a name="outputs--useful-outputs"></a><a name="8.1"></a>
  - [8.2](#outputs--useful-outputs) If you're creating resources with endpoints, DNS names, URLs, etc, include them in the output section. This will save time later. 



    ```yaml
    # bad
    Outputs: 

    # goodOutputs: 
    Outputs:
        auroraClusterEndpoint:
            Description: aurora DNS Endpoint
            Value: !GetAtt auroraCluster.Endpoint.Address

    ```

## Stack Architecture

  <a name="architecture--nesting"></a><a name="9.1"></a>
  - [9.1](#architecture--nesting) Unless absolutely necessary, do not create nested stacks. They are more trouble than they are worth. 

  <a name="architecture--resource-order"></a><a name="9.2"></a>
  - [9.2](#architecture--resource-order) Your resources should appear in the following order: 

```yaml
AWSTemplateFormatVersion: "version date"
Description:
Parameters:
Metadata:
Mappings:
Conditions:
Resources:
Outputs:

```

## Attribution

AWS and CloudFormation are trademarks of AWS <https://aws.amazon.com/trademark-guidelines/>.
