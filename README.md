# Nitor Deploy tools - AWS Project template

Template project for projects that use Amazon Web Service (AWS) as an
environment and Nitor Deploy Tools (NDT) for tooling. This repository
is a template that can be used to start new AWS infra project into own
account.

## Overview

Folders in root level describe components in the virtual private cloud
(VPC). Each folder contains different stacks that are baked into image
that is defined in the component root level (`image` sub folder).

Properties files are applied to environment in order of least to most
specific, i.e. project root is first and stack-branch properties
last. Naturally, last one wins.

This repository contains bootstrapping scripts and manuals that one
can 'quickly' setup a project components into AWS.

### Overview of required properties and existing configuration files

TODO

## Setup

Setuping contains item that needs to created manually into AWS that
tooling can be bootstrapped, ndt tooling setup and intializing project
stacks that one can start implement project specific components.

Prequisite:
  * python < 3.0
  * `nitor-deploy-tools` (in pip)
  * `ansible` (in pip)

Protip: add following shell function to suitable place, to get
autocompletion working.

```shell
if command -v nitor-dt-register-complete > /dev/null 2>&1; then
    eval "$(nitor-dt-register-complete)"
fi
```

### Setup new AWS account

Project template is designed to deploy the technology stack into
allready created AWS account with user that has administration
priviliges.

 1. Create AWS account (optional)
 2. Create IAM user with AdministrativeAccess permissions. *Remember
    to gather access key secrets.*
 3. Setup ndt with command `ndt cli-setup`
 4. Now you should have project profile sourceable in `~/bin/<profile-name>`

### Setup nesessary stacks

NDT provides bootstraping scripts to generate required stacks for
deploying rest of the components.

  1. create network stack with command `ndt setup-networks` ?

### Setup project Jenkins

Project template contains jenkins component which role is to act as an
baking platform for the other components in the pool. It will provide
required CI tools that are needed to bake rest of the component that
are defined in the project.

  1. Reserve a one elastic IP to be assigned for jenkins
  2. Create needed baking roles into project with following command
     `ndt bootstrap bakery-roles??`
  3. Create necessary keypair for accessing instance(s)
  4. TODO do some secrets magic `ndt setup-fetch-secrets` ? Don't
     remember why...
  5. Bake jenkins itself with `ndt bake-image jenkins jenkins-bakery`
     (where jenkins refers to component and jenkins-bakery to the
     stack)
  6. After baking we can deploy the jenkins stack with command `ndt
     deploy-stack jenkins jenkins-bakery`
  7. Now you should go and assign reserved elastic IP into newly
     created instance
  8. TODO Go and install required tools for baking?

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
'scriptApproval' section in the jenkins.

After first succesful run of the bake recipies generation job, jenkins
will have view named `JENKINS_JOB_PREFIX`. Each branch that wanted to
be built must have corresponding`infra-branchname.properties` file
present.

Now changes into project files will trigger modification job that
modify created jobs, meaning that history of the job is presevered.

Following these instructions you should have succesfully bootstrapped
your project, such that it should be possible to bake images from
different branches and deploy them into AWS.
