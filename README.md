# Sceptre

[![Bugs](https://sonarcloud.io/api/project_badges/measure?project=Sceptre_sceptre-cli&metric=bugs)](https://sonarcloud.io/dashboard?id=Sceptre_sceptre-cli)
[![Coverage](https://sonarcloud.io/api/project_badges/measure?project=Sceptre_sceptre-cli&metric=coverage)](https://sonarcloud.io/dashboard?id=Sceptre_sceptre-cli)
[![Maintainability Rating](https://sonarcloud.io/api/project_badges/measure?project=Sceptre_sceptre-cli&metric=sqale_rating)](https://sonarcloud.io/dashboard?id=Sceptre_sceptre-cli)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=Sceptre_sceptre-cli&metric=alert_status)](https://sonarcloud.io/dashboard?id=Sceptre_sceptre-cli)
[![Reliability Rating](https://sonarcloud.io/api/project_badges/measure?project=Sceptre_sceptre-cli&metric=reliability_rating)](https://sonarcloud.io/dashboard?id=Sceptre_sceptre-cli)
[![Security Rating](https://sonarcloud.io/api/project_badges/measure?project=Sceptre_sceptre-cli&metric=security_rating)](https://sonarcloud.io/dashboard?id=Sceptre_sceptre-cli)
[![Technical Debt](https://sonarcloud.io/api/project_badges/measure?project=Sceptre_sceptre-cli&metric=sqale_index)](https://sonarcloud.io/dashboard?id=Sceptre_sceptre-cli)
[![Vulnerabilities](https://sonarcloud.io/api/project_badges/measure?project=Sceptre_sceptre-cli&metric=vulnerabilities)](https://sonarcloud.io/dashboard?id=Sceptre_sceptre-cli)
![image](https://circleci.com/gh/Sceptre/sceptre.png?style=shield)
![image](https://badge.fury.io/py/sceptre.svg)

# Install

`$ pip install sceptre-cli`

More information on installing sceptre can be found in our
[Installation Guide](https://sceptre.cloudreach.com/latest/docs/install.html)

# Use Docker Image

View our [Docker repository](https://hub.docker.com/r/cloudreach/sceptre).
Images available from version 2.0.0 onward.

To use our Docker image follow these instructions:

1. Pull the image `docker pull cloudreach/sceptre:[SCEPTRE_VERSION_NUMBER]` e.g.
   `docker pull cloudreach/sceptre:2.1.4`. Leave out the version number if you
   wish to run `latest` or run `docker pull cloudreach/sceptre:latest`.

2. Run the image. You will need to mount the working directory where your
   project resides to a directory called `project`. You will also need to mount
   a volume with your AWS config to your docker container. E.g.

`docker run -v $(pwd):/project -v /Users/me/.aws/:/root/.aws/:ro cloudreach/sceptre:latest --help`

If you want to use a custom ENTRYPOINT simply amend the Docker command:

`docker run -ti --entrypoint='' cloudreach:latest sh`

The above command will enter you into the shell of the Docker container where
you can execute sceptre commands - useful for development.

If you have any other environment variables in your non-docker shell you will
need to pass these in on the Docker CLI using the `-e` flag. See Docker
documentation on how to achieve this.

# Example

Sceptre organises Stacks into "Stack Groups". Each Stack is represented by a
YAML configuration file stored in a directory which represents the Stack Group.
Here, we have two Stacks, `vpc` and `subnets`, in a Stack Group named `dev`:

```
$ tree
.
├── config
│   └── dev
│        ├── config.yaml
│        ├── subnets.yaml
│        └── vpc.yaml
└── templates
    ├── subnets.py
    └── vpc.py
```

We can create a Stack with the `create` command. This `vpc` Stack contains a
VPC.

```
$ sceptre create dev/vpc.yaml

dev/vpc - Creating stack dev/vpc
VirtualPrivateCloud AWS::EC2::VPC CREATE_IN_PROGRESS
dev/vpc VirtualPrivateCloud AWS::EC2::VPC CREATE_COMPLETE
dev/vpc sceptre-demo-dev-vpc AWS::CloudFormation::Stack CREATE_COMPLETE
```

The `subnets` Stack contains a subnet which must be created in the VPC. To do
this, we need to pass the VPC ID, which is exposed as a Stack output of the
`vpc` Stack, to a parameter of the `subnets` Stack. Sceptre automatically
resolves this dependency for us.

```
$ sceptre create dev/subnets.yaml
dev/subnets - Creating stack
dev/subnets Subnet AWS::EC2::Subnet CREATE_IN_PROGRESS
dev/subnets Subnet AWS::EC2::Subnet CREATE_COMPLETE
dev/subnets sceptre-demo-dev-subnets AWS::CloudFormation::Stack CREATE_COMPLETE
```

Sceptre implements meta-operations, which allow us to find out information about
our Stacks:

```
$ sceptre list resources dev/subnets.yaml

- LogicalResourceId: Subnet
  PhysicalResourceId: subnet-445e6e32
  dev/vpc:
- LogicalResourceId: VirtualPrivateCloud
  PhysicalResourceId: vpc-c4715da0
```

Sceptre provides Stack Group level commands. This one deletes the whole `dev`
Stack Group. The subnet exists within the vpc, so it must be deleted first.
Sceptre handles this automatically:

```
$ sceptre delete dev

Deleting stack
dev/subnets Subnet AWS::EC2::Subnet DELETE_IN_PROGRESS
dev/subnets - Stack deleted
dev/vpc Deleting stack
dev/vpc VirtualPrivateCloud AWS::EC2::VPC DELETE_IN_PROGRESS
dev/vpc - Stack deleted
```

> Note: Deleting Stacks will _only_ delete a given Stack, or the Stacks that are
> directly in a given StackGroup. By default Stack dependencies that are
> external to the StackGroup are not deleted.

Sceptre can also handle cross Stack Group dependencies, take the following
example project:

```
$ tree
.
├── config
│   ├── dev
│   │   ├── network
│   │   │   └── vpc.yaml
│   │   ├── users
│   │   │   └── iam.yaml
│   │   ├── compute
│   │   │   └── ec2.yaml
│   │   └── config.yaml
│   └── staging
│       └── eu
│           ├── config.yaml
│           └── stack.yaml
├── hooks
│   └── stack.py
├── templates
│   ├── network.json
│   ├── iam.json
│   ├── ec2.json
│   └── stack.json
└── vars
    ├── dev.yaml
    └── staging.yaml
```

In this project `staging/eu/stack.yaml` has a dependency on the output of
`dev/users/iam.yaml`. If you wanted to create the Stack `staging/eu/stack.yaml`,
Sceptre will resolve all of it's dependencies, including `dev/users/iam.yaml`,
before attempting to create the Stack.

## CLI

```
Usage: sceptre [OPTIONS] COMMAND [ARGS]...

  Sceptre is a tool to manage your cloud native infrastructure deployments.

Options:
  --version              Show the version and exit.
  --debug                Turn on debug logging.
  --dir TEXT             Specify sceptre directory.
  --output [yaml|json]   The formatting style for command output.
  --no-colour            Turn off output colouring.
  --var TEXT             A variable to template into config files.
  --var-file FILENAME    A YAML file of variables to template into config
                         files.
  --ignore-dependencies  Ignore dependencies when executing command.
  --help                 Show this message and exit.

Commands:
  create         Creates a stack or a change set.
  delete         Deletes a stack or a change set.
  describe       Commands for describing attributes of stacks.
  estimate-cost  Estimates the cost of the template.
  execute        Executes a Change Set.
  generate       Prints the template.
  launch         Launch a Stack or StackGroup.
  list           Commands for listing attributes of stacks.
  new            Commands for initialising Sceptre projects.
  set-policy     Sets Stack policy.
  status         Print status of stack or stack_group.
  update         Update a stack.
  validate       Validates the template.
```

Full API reference documentation can be found in the
[Documentation](https://sceptre.cloudreach.com/)

## Tutorial and Documentation

- [Get Started](https://sceptre.cloudreach.com/latest/docs/get_started.html)
- [Documentation](https://sceptre.cloudreach.com/)

## Contributing

See our [Contributing Guide](CONTRIBUTING.md)
