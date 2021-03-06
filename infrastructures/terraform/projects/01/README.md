# Deploy a Complex Infrastructure in AWS

This project will deploy an end to end infrastructure including the below resources.

- Networking
  - VPC
  - 3 Public subnets
  - 3 Private subnets
  - 3 Ephemeral subnets
    - Ephemeral nodes (nodes that are recycled often like ASG nodes), need to be in separate subnets from long-running nodes (like ElastiCache and RDS) because AWS maintains an ARP cache with a semi-long expiration time. So if node A with IP 10.0.0.123 gets terminated, and node B comes in and picks up 10.0.0.123 in a relatively short period of time, the stale ARP cache entry will still be there, so traffic will just fail to reach the new node.
  - NAT
  - OpenVPN
  - Bastion host
- Data
  - RDS (defaults to Postgres)
  - ElastiCache (defaults to Redis)
  - Consul cluster
  - Vault HA with Consul backend
  - RabbitMQ server
- Compute
  - Windows IIS web server with ASP.NET web app using blue/green deploy strategy

## General Setup

Read the [Getting Started](../../../README.md#getting-started) section first. Be sure to follow all instructions closely, many of these steps require pre/post work to be completed or it won't work.

- [Create an Atlas Account](../../../../setup/general.md#create-atlas-account) and save your username as an environment variable
- [Generate an Atlas Token](../../../../setup/general.md#generate-atlas-token) and save your token as an environment variable
- [Generate AWS Keys](../../../../setup/general.md#generate-aws-keys) and save the access key and secret as an environment variables
- [Generate Certs](../../../../setup/general.md#generate-certs) if you haven't already done so
  - `sh gen_cert.sh hashicorp.com example ../../infrastructures/terraform/certs`
- [Generate Keys](../../../../setup/general.md#generate-keys) if you haven't already done so
  - `sh gen_key.sh ../../infrastructures/terraform/keys` or `sh gen_key.sh ../../infrastructures/terraform/keys ~/.ssh/my-existing-private-key.pem` if you have an existing private key you want to use

## Create Base Artifacts with Packer

First read the [Building Images with Packer](../../../../setup/general.md#building-images-with-packer) docs.

Then, follow the [Packer base template docs](../../../../setup/general.md#base-packer-templates) to run the below commands.

Remember, all `packer push` commands must be performed in the base [`infrastructures`](../../../.) directory.

    $ packer push packer/aws/ubuntu/base.json
    $ packer push packer/aws/windows/base.json

## Create Child Artifacts with Packer

After your base artifacts have been created, push the rest of your Packer templates that depend on them.

Follow the [Packer child template docs](../../../../setup/general.md#child-packer-templates) to run the below commands.

Remember, all `packer push` commands must be performed in the base [`infrastructures`](../../../.) directory.

    $ packer push packer/aws/ubuntu/consul.json
    $ packer push packer/aws/ubuntu/vault.json
    $ packer push packer/aws/ubuntu/rabbitmq.json
    $ packer push packer/aws/windows/web.json

If you decide to update any of the artifact names, be sure those name changes are reflected in [terraform.tfvars](terraform.tfvars#L36-L47).

## Upload your Web Application to Atlas

Follow the [Upload Applications](../../../../setup/general.md#upload-applications) docs to upload and link your application to an associated Build Template following the below steps.

This example assumes you're using the [GitHub Integration](https://atlas.hashicorp.com/help/applications/uploading#github). If you are unable to do so, you can alternatively use [Vagrant Push](https://atlas.hashicorp.com/help/applications/uploading#vagrant-push) or the [Atlas Upload CLI](https://atlas.hashicorp.com/help/applications/uploading#upload-cli).

- Create an uncompiled [web](../../../apps/asp.net/web) "Application" named `asp.net-web`
- Link to the `aws-windows-web` Build Template in "Settings"
- Using the [GitHub Integration](../../../../setup/general.md#github-integration), select your GitHub repository, then enter `infrastructures/apps/asp.net/web` for "Application directory" leaving "Application Template" blank

Upload new versions by merging a commit into master from the [web](../../../apps/asp.net/web) directory. This will upload your latest app code and trigger a Packer build to create a new artifact.

## Provision Infrastructure with Terraform

Follow the [Deploy with Terraform docs](../../../../setup/general.md#deploy-with-terraform) to run the below commands.

From the base `infrastructures` directory, navigate to `terraform/projects/01/.`

    $ cd terraform/projects/01

If this is the first time you have run Terraform in this project, you will need to setup the remote config and download the modules.

If you updated the `atlas_environment` variable in [`terraform.tfvars`](terraform.tfvars#L17) from `example-01`, be sure that change is reflected in the below `terraform remote config` and `terraform push` commands.

    $ terraform remote config -backend-config name=$ATLAS_USERNAME/example-01
    $ terraform get

If everything looks good, run

    $ terraform push -name $ATLAS_USERNAME/example-01 -var "atlas_token=$ATLAS_TOKEN" -var "atlas_username=$ATLAS_USERNAME"

This takes about 20 minutes to run as RDS takes quite awhile to provision. Don't forget to `terraform destroy` you're environment when your done so you don't rack up AWS bills (unless you plan on keeping it around).

That's it!
