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

## Signing a Container

Utilize the following steps to sign a container that has been published to an OCI registry

1. Add the root CA that was created to your local truststore. The certificate can be obtained by navigating to `https://rekor.<base_domain>` in a browser.

2. Export the following environment variables substituting `base_hostname` with the value used as part of the provisioning

```shell
export COSIGN_EXPERIMENTAL=1
export KEYCLOAK_REALM=sigstore
export BASE_HOSTNAME=<base_hostname>
export FULCIO_URL=https://fulcio.$BASE_HOSTNAME
export KEYCLOAK_URL=https://keycloak.$BASE_HOSTNAME
export REKOR_URL=https://rekor.$BASE_HOSTNAME
export TUF_URL=https://tuf.$BASE_HOSTNAME
export KEYCLOAK_OIDC_ISSUER=$KEYCLOAK_URL/realms/$KEYCLOAK_REALM
```

3. Initialize the TUF roots

```shell
cosign initialize --mirror=$TUF_URL --root=$TUF_URL/root.json
```

Note: If you have used `cosign` previously, you may need to delete the `~/.sigstore` directory

4. Sign the desired container

```shell
cosign sign --force -y --fulcio-url=$FULCIO_URL --rekor-url=$REKOR_URL --oidc-issuer=$KEYCLOAK_OIDC_ISSUER  <image>
```

Authenticate with the Keycloak instance using the desired credentials.

5. Verify the signed image

```shell
cosign sign --force -y --fulcio-url=$FULCIO_URL --rekor-url=$REKOR_URL <image>
```

If the signature verification did not result in an error, the deployment of Sigstore was successful!

## Future Efforts

The following are planned next steps:

- [ ] Replace Sigstore OIDC with Keycloak
- [ ] Update `Pod` manifests to `Deployment` manifests
- [ ] Configure pods as systemd services

## Feedback

Any and all feedback is welcomed. Submit an [Issue](https://github.com/sabre1041/sigstore-ansible/issues) or [Pull Request](https://github.com/sabre1041/sigstore-ansible/pulls) as desired.
