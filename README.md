# Nitor Deploy tools - AWS Project template

Template project for projects that use Amazon Web Service (AWS) as an
environment and Nitor Deploy Tools (NDT) for tooling. This repository
is a template that can be used to start new AWS infra project into own
account.

## Overview

![overview](docs/figs/overview.png)

Folders in root level describe components in the virtual private cloud
(VPC). Each folder contains different stacks that are baked into image
that is defined in the component root level (`image` sub folder).

Properties files are applied to environment in order of least to most
specific, i.e. project root is first and stack-branch properties
last. Naturally, last one wins.

This repository contains bootstrapping scripts and manuals that one
can 'quickly' setup a project components into AWS.

### Overview of required properties and existing configuration files

This project template has defined only required property files and
required properties into this project. Descriptions of required
properties are distributed into these properties files. To share
custom properties in the component hierarchy, define them in these
files.

Properties in these files should be distributed like the components
are, i.e. component specific properties should be in component
specific properties files. One of the benifits is that it makes
understanding and maintanance of your project infra easier.

## Setup

Setuping contains item that needs to created manually into AWS that
tooling can be bootstrapped, ndt tooling setup and intializing project
stacks that one can start implement project specific components.

Prequisite:
  * `python < 3.0`
  * `nitor-deploy-tools` (in pip)
  * `ansible` (in pip)
    * ansible needs `boto`
  * `jq`

Protip: add following shell function to suitable place, to get
autocompletion working.

```shell
if command -v nitor-dt-register-complete > /dev/null 2>&1; then
    eval "$(nitor-dt-register-complete)"
fi
```

### Setup new AWS account

Project template is designed to deploy the technology stack into
already created AWS account with user that has administration
priviliges.

 1. Create AWS account (if needed)
 2. Create IAM user with AdministrativeAccess permissions. *Remember
    to gather access key secrets.*
 3. Setup ndt with command `ndt setup-cli`
	1. Profile name is id to this account access in your local machine
    2. key ID and secrects are those that you created to in previous step
    3. Default region is in format of AWS labeling (i.e. eu-central-1 etc)
 4. Now you should have project profile sourceable in `~/bin/<profile-name>`

Now you should have programmatic access to your AWS account where the
stack can be deployed.

### Git

NDT is meant to be run at the root of a git repository. So either clone
a repository for this purpose or initialize one in the current directory
by running
``` bash
git init .
```

### Setup bootstrap stacks

NDT provides bootstrapping scripts to generate required stacks for
deploying rest of the components.

#### Network
Create network stack with command `ndt create-stack network`. NDT
proposes defaults that will work unless you have specific routing
needs.

Once ndt has created the stack and other necessary files, ndt will
print instructions on how to deploy the stack. In this case it is:
``` bash
ndt deploy-stack bootstrap network
```

#### Secrets store

You will need a solution for storing secrets like passwords, keys and
certificates. The easiest way to achive this is to use nitor-vault.
Run `vault -i` to create nitor-vault stack into your AWS account.
This creates a CloudFormation stack with a KMS key and an S3 bucket
where one can store and retrieve shared secters.

Taking this to use in all of your stacks and baking requires to have
`fetch-secrets.sh` and `store-secret.sh` on your path that use
vault for the actual storing and fetching. This is easy to set up by
running `sudo setup-fetch-secrets.sh vault`

You have the option of implementing secret storage by yourself. You are
only required to provide the above scripts with the following usage:

``` bash
fetch-secrets.sh get <mode> <secret-paths...>
# Get secret paths with given file mode stored as $(basename secret-path)
fetch-secrets.sh show <secret-name>
# Prints given secret to stdout
fetch-secrets.sh <login|logout>
# Logs in or logs out to underlying secret store (if necessary - if not, you can provide dummy implementations for these calls)
```

``` bash
store-secrets.sh <secret-name>
# Reads secret from stdin and store it with given name
```

#### Bakery roles

Create needed baking roles into project with following command
`ndt create-stack bakery-roles`. Again once required files are created,
you can deploy the stack with the instructions printed at the end of
the command output.

#### DNS

To effectively run services, you need a public DNS name to refer to
the components. There are many ways to set this up, but the instance
templates here require that you set this up outside of this project.
You can delegate a subdomain to Route53 from some other account or DNS
service, you can register a new domain in Route53 or you can even
migrate an existing domain from another service.

Documentation for all of the options here:
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring.html

It is recommended to just register a new domain since it's relatively
cheap and keeps the new infrastructure nicely isolated from any external
dependencies.

Once you have a domain set up, you can set up a common shared file for
easy access for the rest of the templates by running `ndt create-stack route53`

Here there is now stack to actually deploy, only a shared yaml file is
created with the id and name of the hosted zone to use.

#### SSH Keys

There are three main uses for ssh keys in ndt defined architecture:
1. Key used in baking - ansible uses this key to log in an do all necessary configuration
2. Key used in running instances - key that can be used to log in to do troubleshooting
   and (hopefully rare) manual maintenance
3. Key that an insance uses to authenticate to external services like git repositories

All of the keys above could be the same key, but here we show how to make each key separate.

##### Create a key for baking:

``` bash
AWS_KEY_NAME=aws-bake-key
aws ec2 create-key-pair --key-name $AWS_KEY_NAME | jq -r .KeyMaterial > ~/.ssh/$AWS_KEY_NAME.pem`
echo "AWS_KEY_NAME=$AWS_KEY_NAME" >> infra.properties
vault -s -f ~/.ssh/$AWS_KEY_NAME.pem
```
##### Create a key for running instanes:

``` bash
AWS_KEY_NAME=aws-instances
aws ec2 create-key-pair --key-name $AWS_KEY_NAME | jq -r .KeyMaterial > ~/.ssh/$AWS_KEY_NAME.pem`
echo "paramSshKeyName=$AWS_KEY_NAME" >> infra.properties
vault -s -f ~/.ssh/$AWS_KEY_NAME.pem
```
##### Create a key for an instance to identify itself:

The instance starting up will often look (depending on instance needs) for an
ssh key to identify itself from the secret store with the name `hostname.rsa`

``` bash
AWS_KEY_NAME=jenkins.example.com.rsa
ssh-keygen -b 4096 -f ~/.ssh/$AWS_KEY_NAME
vault -s -f ~/.ssh/$AWS_KEY_NAME
vault -s -f ~/.ssh/$AWS_KEY_NAME.pub
```
### Good to go



After successful run, you should now have required stacks (network,
vault, bakery-roles) formed into CloudFormation, stack configurations
generated into `bootstrap` folder, dns zone setup and necessary
instance keys.

### Setup project Jenkins

Project template contains jenkins component which role is to act as an
baking platform for the other components in the pool. It will provide
required CI tools that are needed to bake rest of the component that
are defined in the project.

  1. Reserve a one elastic IP to be assigned for jenkins.
  2. Create accessible route53 hosted zones.
  3. Make sure that domain has certification keys available in vault
     with `CERT_DIR=. ensure-letsencrypt-certs.sh <domain-name>`
  4. Make sure that vault has previously created keypair stored to
	 `<jenkins-domain-name>.rsa` named secret. If not, see instructions above.
  5. Create jenkins stack by running `ndt create-stack jenkins`
  6. Accept the license terms of the base ami used for baking. Even though
     we use a free ami for the base image, the ami license terms need to be
     accepted via the Amazon Marketplace. https://aws.amazon.com/marketplace,
     search for the ami used (value of AMIID_centos in infra.properties),
     click "Continue to subscribe", go to "Manual launch", click 
     "Accept software terms" on the top on the right.
  6. Bake jenkins itself with `ndt bake-image jenkins jenkins`
     (where first jenkins refers to component and latter jenkins to the
     stack).
  7. After baking we can deploy the jenkins stack with command `ndt
     deploy-stack jenkins jenkins "$(cat ami-id.txt)"`
  8. Now you can access Jenkins from the new address. Admin password
     can be retrieved from instance logs
     `/var/log/jenkins/jenkins.log` Login user is centos and the key is
     the one configured for running instances in the instructions above

### Setup components of the stack to bakery

Now that AWS project has been bootsrapped you can start baking and
deploying rest of the components in the stack. For the component to be
deployed one needs to create a new 'freestyle' type of jenkins
job. This job then will take this repository as input and use 'Process
JOB DSLs' plugin to scrape through repository and create views and
jobs for each stack that representend in the repository. Magic is done
in `generate_jobs.groovy` which lays in the root of the project and
will be given as input to the DSL task.

First baking will require permission granting for the job from
'ScriptApproval' section in the jenkins.

After first succesful run of the bake recipies generation job, jenkins
will have view named `JENKINS_JOB_PREFIX`. Each branch that wanted to
be built must have corresponding`infra-branchname.properties` file
present.

Now changes into project files will trigger modification job that
modify created jobs, meaning that history of the job is presevered.

Following these instructions you should have succesfully bootstrapped
your project, such that it should be possible to bake images from
different branches and deploy them into AWS.
