# sigstore-ansible

Automation to deploy the sigstore ecosystem on Virtual Machines

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

## Provision

Execute the following commands to execute the automation:

```shell
ansible-playbook -i inventory playbooks/install.yml
```