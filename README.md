# sigstore-ansible

Automation to deploy the sigstore ecosystem on Virtual Machines

:warning: **The contents of this repository are a Work in Progress.**

## Overview

The automation within this repository establishes the components of the [Sigstore project](https://sigstore.dev) within a single Red Hat Enterprise Linux (RHEL) virtual machine using a standalone containerized deployment. Containers are spawned using Kubernetes based manifests using [podman kube play](https://docs.podman.io/en/latest/markdown/podman-kube-play.1.html).

The following Sigstore components are deployed as part of this architecture:

* [Rekor](https://docs.sigstore.dev/rekor/overview)
    * [Trillian](https://github.com/google/trillian)
* [Fulcio](https://docs.sigstore.dev/fulcio/overview)
* [Certificate Log](https://docs.sigstore.dev/fulcio/certificate-issuing-overview)
* [Keycloak](https://www.keycloak.org)

An [NGINX](https://www.nginx.com) frontend is placed as an entrypoint to the various backend components. Communication is secured via a set of self-signed certificates that are generated at runtime.

[Keycloak](https://www.keycloak.org) is being used as a OIDC issuer for facilitating keyless signing.

Utilize the steps below to understand how to setup and execute the provisioning.

## Prerequisites

Ansible must be installed and configured on a control node that will be used to perform the automation.

NOTE: Future improvements will make use of an Execution environment

Perform the following steps to prepare the control node for execution.

### Dependencies

Install the required Ansible collections by executing the following 

```shell
ansible-galaxy collection install -r requirements.yml 
```

### Inventory

Populate the `sigstore` group within the [inventory](inventory) file with details related to the target host.

### Keycloak

Keycloak is deployed to enable keyless (OIDC) signing. A dedicated realm called `sigstore` is configured by default using a client called `sigstore`

To be able to sign containers, you will need to authenticate to the Keycloak instance. By default, a single user (jdoe) is created. This can be customized by specifying the `keycloak_sigstore_users` variable. The default value is shown below and can be used to authenticate to Keycloak if no modifications are made:

```yaml
keycloak_sigstore_users:
 - username: jdoe
   first_name: John
   last_name: Doe
   email: jdoe@redhat.com
   password: mysecurepassword
```

### Ingress

The automation deploys and configures a software load balancer as a central point of ingress. Multiple hostnames underneath a _base hostname_ are configured and include the following hostnames:

* https://rekor.<base_hostname>
* https://fulcio.<base_hostname>
* https://keycloak.<base_hostname>
* https://tuf.<base_hostname>

Each of these hostnames must be configured in DNS to resolve to the target Virtual Machine. The `base_hostname` parameter must be provided when executing the provisining.

### Cosign

[cosign](https://github.com/sigstore/cosign) is used as part of testing and validating the setup and configuration. It is an optional install if there is not a desire to perform the validation as described below.

## Provision

Execute the following commands to execute the automation:

```shell
ansible-playbook -i inventory playbooks/install.yml -e base_hostname=<base_hostname>
```

### Add the root CA that was created to your local truststore.

The certificate can be downloaded from the browser Certificate Viewer by navigating to `https://rekor.<base_domain>`.
Download the _root_ certiicate that issued the rekor certificate.
In Red Hat based systems, the following commands will add a CA to the system truststore.

```shell
$ sudo openssl x509 -in ~/Downloads/root-cert-from-browser -out sigstore-ca.pem --outform PEM
$ sudo mv sigstore-ca.pem /etc/pki/ca-trust/source/anchors/
$ sudo update-ca-trust
```

## Signing a Container

Utilize the following steps to sign a container that has been published to an OCI registry

1. Export the following environment variables substituting `base_hostname` with the value used as part of the provisioning

```shell
export KEYCLOAK_REALM=sigstore
export BASE_HOSTNAME=<base_hostname>
export FULCIO_URL=https://fulcio.$BASE_HOSTNAME
export KEYCLOAK_URL=https://keycloak.$BASE_HOSTNAME
export REKOR_URL=https://rekor.$BASE_HOSTNAME
export TUF_URL=https://tuf.$BASE_HOSTNAME
export KEYCLOAK_OIDC_ISSUER=$KEYCLOAK_URL/realms/$KEYCLOAK_REALM
```

2. Initialize the TUF roots

```shell
cosign initialize --mirror=$TUF_URL --root=$TUF_URL/root.json
```

Note: If you have used `cosign` previously, you may need to delete the `~/.sigstore` directory

3. Sign the desired container

```shell
cosign sign -y --fulcio-url=$FULCIO_URL --rekor-url=$REKOR_URL --oidc-issuer=$KEYCLOAK_OIDC_ISSUER  <image>
```

Authenticate with the Keycloak instance using the desired credentials.

4. Verify the signed image

Refer to this example that verifies an image signed with email identity `sigstore-user@email.com` and issuer `https://github.com/login/oauth`.

```shell
cosign verify \
--rekor-url=$REKOR_URL \
--certificate-identity-regexp sigstore-user \
--certificate-oidc-issuer-regexp keycloak  \
<image>
```

If the signature verification did not result in an error, the deployment of Sigstore was successful!

## Execution Environments support

This deployment can be run inside an [Ansible Execution Environment](https://docs.ansible.com/automation-controller/latest/html/userguide/execution_environments.html).
To build an Execution Environment and run this deployment inside a custom container, reproduce the following steps:

1. Populate the `execution-environment.yml` file with the base image to use as a value for the `EE_BASE_IMAGE` variable in `build_arg_defaults`. For more information on how to write an `execution-environment.yml` file with more available options, refer to the following [article](https://www.redhat.com/sysadmin/ansible-execution-environment-unconnected#:~:text=Ansible%20execution%20environments%20(EE)%20were,that%20help%20execute%20Ansible%20playbooks.)

2. If `ansible-builder` is not present on your environment, install it by using `python3 -m pip install ansible-builder`. Run the `ansible-builder build --tag my_ee` to build the Execution Environment. This command will create a directory `context/` with `_build` information about requirements for the image and a Containerfile

3. Ensure that you are logged in the target container image registry, then tag and push the created image to the registry:

```
docker tag my-ee:latest quay.io/myusername/my_ee:latest
docker push quay.io/myusername/my_ee:latest
```

4. To run this deployment inside the created Execution Environment, use the [`ansible-navigator`](https://ansible-runner.readthedocs.io/en/stable/) command line. It can be installed via the `python3 -m pip install ansible-navigator` command or refer to the [installation instructions](https://ansible-navigator.readthedocs.io/en/latest/installation/#installing-ansible-navigator-with-execution-environment-support). `ansible-navigator` supports `ansible-playbook` commands to run automation jobs, but adds more capabilities like Execution Environment support.

5. Create an `env/` directory at the root of the repository, and create an `env/extravars` file. Populate it with your base hostname as follows:

```
---
base_hostname: base_hostname
```

6. Run the deployment job inside the Execution Environment:

```
ansible-navigator run -i inventory --execution-environment-image=quay.io/myusername/my-ee:latest playbooks/install.yml
```

## Future Efforts

The following are planned next steps:

- [ ] Update `Pod` manifests to `Deployment` manifests
- [ ] Configure pods as systemd services

## Feedback

Any and all feedback is welcomed. Submit an [Issue](https://github.com/sabre1041/sigstore-ansible/issues) or [Pull Request](https://github.com/sabre1041/sigstore-ansible/pulls) as desired.
