# Nitor Deploy tools - Infra Project template

Template project for projects that use Amazon Web Service (AWS) as an
environment and ndt tooling. This repository is a template that can be
used to start new AWS infra.

## Overview

Folders in root level describe components in the virtual private cloud
(VPC). Each folder contains different stacks that are baked into image
that is defined in the component root level.

Properties files are applied to environment in order of least to most
specific, i.e. project root is first and stack-branch properties
last. Naturally, last one wins.

Project is supposed to be integrated into bakery that will
autogenerate jobs that will bake different stacks and enable easy
deployment and undeployment.

## Setup

To integrate project into Nitor ami bakery, one needs to go
https://amibakery.nitor.zone/ and create new freestyle type project
which will run "Process JOB DSLs" task with `generate_jobs.groovy` as
its input. Action for removed jobs should be 'delete'.

First baking will require permission granting for the job from
https://amibakery.nitor.zone/scriptApproval/.

After first succesful run of the bake recipies generation job, jenkins
will have view named `JENKINS_JOB_PREFIX`. Each branch that wanted to
be built must have corresponding`infra-branchname.propertis` file
present.

Now changes into project configurations or `generate_jobs.goovy` file
will trigger modification that will change affected generated jobs
accordingly.
