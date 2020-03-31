**Note**: This public repo contains the documentation for the private GitHub repo <https://github.com/gruntwork-io/module-security>.
We publish the documentation publicly so it turns up in online searches, but to see the source code, you must be a Gruntwork customer.
If you're already a Gruntwork customer, the original source for this file is at: <https://github.com/gruntwork-io/module-security/blob/master/codegen/core-concepts.md>.
If you're not a customer, contact us at <info@gruntwork.io> or <http://www.gruntwork.io> for info on how to get access!

# Generate AWS Config Core Concepts


## Installation

Each generator is a single, self-contained, statically compiled binary written in Go. The easiest way to get it onto
your servers is to use the [Gruntwork Installer](https://github.com/gruntwork-io/gruntwork-installer). For example, to
install `generate-aws-config` (make sure to replace `<VERSION>` below with the latest version from the [releases
page](https://github.com/gruntwork-io/module-security-public/releases)):

```
gruntwork-install --binary-name generate-aws-config --repo https://github.com/gruntwork-io/module-security --tag <VERSION>
```

Alternatively, you can download the binary from the [Releases
Page](https://github.com/gruntwork-io/module-security-public/releases).


## Usage

The generator command should be run in your `infrastructure-modules` repository, or any repository where you store your
Terraform modules. This command will output a Terraform module that you can then deploy with various input parameters to
enable the single region module on all regions.

For example, `generate-aws-config` will generate a Terraform module named `aws-config-multi-region` which exposes input
parameters to adjust various options exposed in the single region module. Suppose you had an `infrastructure-modules`
repository that mimics the Gruntwork Reference Architecture, and has the following structure:

```
infrastructure-modules
├── README.md
├── data-stores
│   └── rds
│       ├── README.md
│       ├── main.tf
│       ├── outputs.tf
│       └── vars.tf
└── security
    ├── iam-groups
    │   ├── README.md
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── vars.tf
    └── cloudtrail
        ├── README.md
        ├── main.tf
        ├── outputs.tf
        └── vars.tf
```

Suppose that you now want to add the module `aws-config-multi-region` as generated by this script in the `security`
folder. To do so, first install the binary so that it is available in your `PATH` as described in the [Installation
section of this docs](#installation).

Next, run the generator in the `infrastructure-modules` directory, passing in the target directory and main region. Note
that you will need to be authenticated to your AWS account. Refer to the [Comprehensive Guide to Authenticating to AWS
on the Command
Line](https://blog.gruntwork.io/a-comprehensive-guide-to-authenticating-to-aws-on-the-command-line-63656a686799) for
recommended ways to authenticate on the command line.

Note that one region must be marked as the one to store the global recorder, which includes resources that don't have
regions in AWS such as IAM.

```
generate-aws-config --target-directory ./security/aws-config --global-recorder-region us-east-1
```

This will:

- Look up all the available regions for the authenticated account.
- Generate a terraform module in the folder `security/aws-config` that will deploy AWS Config on each enabled region.
- Set the us-east-1 as the one to store the global recorder.

At the end of this command, you should see the `aws-config` module generated in your `infrastructure-modules`
repository:

```
infrastructure-modules
├── README.md
├── data-stores
│   └── rds
│       ├── README.md
│       ├── main.tf
│       ├── outputs.tf
│       └── vars.tf
└── security
    ├── aws-config
    │   ├── README.md
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── vars.tf
    ├── iam-groups
    │   ├── README.md
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── vars.tf
    └── cloudtrail
        ├── README.md
        ├── main.tf
        ├── outputs.tf
        └── vars.tf

```

You can now invoke the module using Terragrunt, or any other production Terraform frontend that you are currently using.


## Building the Binary

We use [packr](https://github.com/gobuffalo/packr) to compile the templates into the binary so that it is portable. To
do so, we need to run `packr2` prior to building the binaries.

Here are the steps for compiling from source:

- Install the `packr2` binary: `go get -u github.com/gobuffalo/packr/v2/packr2`
- Run `$GOPATH/bin/packr2`. This will convert the template files into go files so that they are available in the go
  binary.
- Build the `generate-aws-config` binary: `go build -o $BIN_PATH .`


## Alternatives considered

### Native Terraform

As of 0.12.16, Terraform still does not support:
 * [for_each or counts with modules](https://github.com/hashicorp/terraform/issues/17519)
 * [for_each or counts with providers](https://github.com/hashicorp/terraform/issues/19932)
 * [interpolating variables on the provider parameter of a resource](https://github.com/hashicorp/terraform/issues/18682)

The combination of these limitations makes it impossible to natively write Terraform code that invokes the [aws-config
module](https://github.com/gruntwork-io/module-security-public/tree/master/modules/aws-config) across all regions.

### Manually generating the code

If this tool didn't exist, users will have to manually create the module with each of the enabled regions replicated.
This involves a lot of copy paste and with 10+ regions, it becomes prone to operator error. Additionally, there are
several manual steps required:

1. The user will have to use the aws CLI to look up all enabled regions.
1. For each enabled region, the user will have to add in the provider config and module call.
1. One of the regions need to be designated the global recorder and have the input `is_global_recorder = true`.

This is especially painful if the module needs to be updated, as all references need to be replaced. When the config is
almost exactly the same (except for the region and the global recorder), this can be unnecessarily painful to replicate
the changes in each block.

### Generating Terragrunt live config

Instead of generating a single module that manages all the configs, we could also have generated Terragrunt live config
files for each region in the Terragrunt folder structure. Ultimately we decided to generate the module calls in
Terraform for the following reasons:

- We don't want to be opinionated to Terragrunt and support a wide range of possible use cases including Terraform
  Enterprise.
- Removing a region requires running `terragrunt destroy`. If you forget to run `terragrunt destroy` on the removed
  region, then you may never remove that Config until you manually remove it from the console. In contrast, the current
  approach will eventually ensure the Config is removed when `terraform apply` is run after the code for the region is
  removed.

Additionally, replicating the Terraform module blocks in a single module has a natural progression to migrate to using
`for_each` on modules once that is ready to be implemented.

### Supporting iteration in Terragrunt

This ended up being too complex to justify supporting the feature. See the [RFC for Terragrunt
iteration](https://github.com/gruntwork-io/terragrunt/pull/853) for more information.