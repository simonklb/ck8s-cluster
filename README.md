Elastisys Compliant Kubernetes Cluster
======================================

**NOTE: This project is in alpha stage and is actively developed.
Therefore the API may change in backwards-incompatible ways.**

## Overview

TODO

### Cloud providers

Currently we support three cloud providers: Exoscale, Safespring, an CityCloud.

### Requirements

- [BaseOS](https://github.com/elastisys/ck8s-base-vm) (tested with 0.0.6)
- [terraform](https://www.terraform.io/downloads.html) (tested with 0.12.19)
- [kubectl](https://github.com/kubernetes/kubernetes/releases) (tested with 1.15.2)
- [jq](https://github.com/stedolan/jq) (tested with jq-1.5-1-a5b5cbe)
- [sops](https://github.com/mozilla/sops) (tested with 3.6.1)
- [s3cmd](https://s3tools.org/s3cmd) (tested with 2.0.2)
- [ansible](https://www.ansible.com) (tested with 2.5.1)
- [go](https://golang.org) (tested with 1.13.8)

Note that you will need a [BaseOS](https://github.com/elastisys/ck8s-base-vm) VM template available at your cloud provider of choice!
See the [releases](https://github.com/elastisys/ck8s-base-vm/releases) for available VM images that can be uploaded to the cloud provider.

### Terraform Cloud

The Terraform state is stored in the [Terraform Cloud remote backend](https://www.terraform.io/docs/backends/types/remote.html).
If you haven't done so already, you first need to:

1. [Create an account](https://app.terraform.io/signup/account)

2. Add your [authentication token](https://app.terraform.io/app/settings/tokens) in the `.terraformrc` file.
[Read more here](https://www.terraform.io/docs/enterprise/free/index.html#configure-access-for-the-terraform-cli).

### PGP

Configuration secrets in ck8s are encrypted using [SOPS](https://github.com/mozilla/sops).
We currently only support using PGP when encrypting secrets.
Because of this, before you can start using ck8s, you need to generate your own PGP key:

```bash
gpg --full-generate-key
```

Note that it's generally preferable that you generate and store your primary key and revocation certificate offline.
That way you can make sure you're able to revoke keys in the case of them getting lost, or worse yet, accessed by someone that's not you.

Instead create subkeys for specific devices such as your laptop that you use for encryption and/or signing.

If this is all new to you, here's a [link](https://riseup.net/en/security/message-security/openpgp/best-practices) worth reading!

## Usage

### Quickstart

To build the cli simply run the following:

```
make build
```

The binary can then be found in `dist/ck8s_linux_amd64`.
You can now move (or create a link to) this binary to a location in your PATH and rename it to `ck8s`.
If you don't, you'll need to replace all the commands referred to as `ck8s` to be `dist/ck8s_linux_amd64`.

In order to setup a new Compliant Kubernetes cluster you will need to do the following:

Set the path of your configuration either as the environment variable `CK8S_CONFIG_PATH` or use the flag `--config-path`.
You also need to set the path to the code which you can do by setting the environment variable `CK8S_CODE_PATH` or use the flag `--code-path` (it defaults to `./` so you don't need to set it if you are running it from the repo root).
These options are needed for all commands, so it's often recommended to set them as environment variable.

Set the fingerprint of your PGP-key to either `CK8S_PGP_FP` or use the `--pgp-fp` flag.

Then run the following:

**NOTE:** To not cause any confusion from the old cli, we decided to "hard deprecate" the environment variables `CK8S_PGP_UID`, `CK8S_ENVIRONMENT_NAME` and `CK8S_CLOUD_PROVIDER` so make sure those are not set.

```
ck8s init <environment name> <cloud provider> [--pgp-fp <PGP key fingerprint>] [--config-path <config path>] [--code-path <path to repo root>]
```

This will create some files that you need to edit to make it work.
The minimum requirements is that you edit `${CK8S_CONFIG_PATH}/tfvars.json` to include your IP address in the whitelists and that you add your credentials to the sops encrypted file `${CK8S_CONFIG_PATH}/secrets.yaml`.
See [here](#configuration) for more information

Note that if there already exists a terraform workspace with the same name as your environment name, then you may need to destroy it  before you continue to the next step.
You can remove the workspace either through the terraform cli or via the backend it is stored in.

Make sure you are logged in to terraform cli (or that you have a valid token in `~/.terraformrc`) and run:

```
ck8s apply --cluster sc
ck8s apply --cluster wc
```

The cluster should now be up and running. You can verify this with:

```
ck8s status --cluster sc
ck8s status --cluster wc
```

### Completion

To enable completion you can source the code generated by running `ck8s completion <shell>`.
See `ck8s completion --help` for more details and examples.

### Configuration

Some configurations do not have default values and needs to be set before the cluster can be created.
These are the values that needs to be provided by you

#### `backend.hcl`

* `Organization`: The organization to use in Terraform Cloud

#### `config.yaml`

*Citycloud/Safespring only*

* `os_project_domain_name`: Openstack project domain name to use
* `os_project_id`: Openstack project ID to use
* `os_user_domain_name`: Openstack user domain name to use

#### `tfvars.json`

* `public_ingress_cidr_whitelist`: IP whitelist of ssh port
* `api_server_whitelist`: IP whitelist of api server
* `nodeport_whitelist`: IP whitelist of the nodeports (30000-32767)

*Citycloud/Safespring only*

* `aws_dns_zone_id`: The AWS DNS zone ID to use for DNS entries
* `aws_dns_role_arn`: The AWS role to assume when creating DNS entry

#### `secrets.yaml`

* `s3_access_key`: Access key to S3 (For storing blobs)
* `s3_secret_key`: Secret key to S3

*Exoscale only*

* `exoscale_api_key`: API key to exoscale
* `exoscale_secret_key`: Secret key to exoscale

*Citycloud/Safespring only*

* `os_username`: Openstack username
* `os_password`: Openstack password
* `aws_access_key_id`: AWS access key ID (for DNS)
* `aws_secret_access_key`: AWS secret key

### DNS

The domain name will be automatically created with the name `${TF_VAR_dns_prefix}[.ops].a1ck.io` for Exoscale and `${TF_VAR_dns_prefix}[.ops].elastisys.se` for Safespring and Citycloud.
In Exoscale we use Exoscale's own DNS features while for Safespring and Citycloud we use AWS.

For Safespring and Citycloud the domain can be changed by setting the Terraform variable `aws_dns_zone_id` to an id of another hosted zone in AWS Route 53.

## Development

When developing the cli the most convenient way of running the cli is:

```
go run ./cmd/ck8s
```
