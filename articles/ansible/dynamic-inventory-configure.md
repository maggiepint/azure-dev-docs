---
title: Tutorial - Configure dynamic inventories of your Azure resources using Ansible
description: Learn how to use Ansible to manage your Azure dynamic inventories
keywords: ansible, azure, devops, bash, cloudshell, dynamic inventory
ms.topic: tutorial
ms.date: 10/30/2020
ms.custom: devx-track-ansible, devx-track-azurecli
---

# Tutorial: Configure dynamic inventories of your Azure resources using Ansible

[!INCLUDE [ansible-28-note.md](includes/ansible-28-note.md)]

Ansible can be used to pull inventory information from various sources (including cloud sources such as Azure) into a *dynamic inventory*. 

[!INCLUDE [ansible-tutorial-goals.md](includes/ansible-tutorial-goals.md)]

> [!div class="checklist"]
>
> * Configure two test virtual machines.
> * Tag one of the virtual machines
> * Install Nginx on the tagged virtual machines
> * Configure a dynamic inventory that includes the configured Azure resources

## Prerequisites

[!INCLUDE [open-source-devops-prereqs-azure-subscription.md](../includes/open-source-devops-prereqs-azure-subscription.md)]
[!INCLUDE [open-source-devops-prereqs-create-service-principal.md](../includes/open-source-devops-prereqs-create-service-principal.md)]
[!INCLUDE [ansible-prereqs-cloudshell-use-or-vm-creation2.md](includes/ansible-prereqs-cloudshell-use-or-vm-creation2.md)]

## Create the test VMs

1. Sign in to the [Azure portal](https://go.microsoft.com/fwlink/p/?LinkID=525040).

1. Open [Cloud Shell](/azure/cloud-shell/overview).

1. Create an Azure resource group to hold the virtual machines for this tutorial.

    > [!IMPORTANT]
    > The Azure resource group you create in this step must have a name that is entirely lower-case. Otherwise, the generation of the dynamic inventory will fail.

    ```azurecli
    az group create --resource-group ansible-inventory-test-rg --location eastus
    ```

1. Create two Linux virtual machines on Azure using one of the following techniques:

    - **Ansible playbook** - The article, [Create a basic virtual machine in Azure with Ansible](./vm-configure.md) illustrates how to create a virtual machine from an Ansible playbook. If you use a playbook to define one or both of the virtual machines, ensure that the SSH connection is used instead of a password.

    - **Azure CLI** - Issue each of the following commands in the Cloud Shell to create the two virtual machines:

        ```azurecli
        az vm create --resource-group ansible-inventory-test-rg \
                     --name ansible-inventory-test-vm1 \
                     --image UbuntuLTS --generate-ssh-keys
        ```

        ```azurecli
        az vm create --resource-group ansible-inventory-test-rg \
                     --name ansible-inventory-test-vm2 \
                     --image UbuntuLTS --generate-ssh-keys
        ```

## Tag a VM

You can [use tags to organize your Azure resources](/azure/azure-resource-manager/resource-group-using-tags#azure-cli) by user-defined categories.

Enter the following [az resource tag](/cli/azure/resource#az_resource_tag) command to tag the virtual machine `ansible-inventory-test-vm1` with the key `Ansible=nginx`:

```azurecli
az resource tag --tags Ansible=nginx --id /subscriptions/<YourAzureSubscriptionID>/resourceGroups/ansible-inventory-test-rg/providers/Microsoft.Compute/virtualMachines/ansible-inventory-test-vm1
```

Replace `<YourAzureSubscriptionID>` with your SubscriptionID.

## Generate a dynamic inventory

Once you have your virtual machines defined (and tagged), it's time to generate the dynamic inventory.

Ansible provides an [Azure dynamic-inventory plug-in](https://github.com/ansible/ansible/blob/stable-2.9/lib/ansible/plugins/inventory/azure_rm.py). The following steps walk you through using the plug-in:

1. The dynamic inventory must end in `azure_rm` and have an extension of either `yml` or `yaml` otherwise Ansible will not detect the proper inventory plugin. For this tutorial example, save the following playbook as `myazure_rm.yml`:

    ```yml
        plugin: azure_rm
        include_vm_resource_groups:
        - ansible-inventory-test-rg
        auth_source: auto
    
        keyed_groups:
        - prefix: tag
          key: tags
    ```

1. Run the following command to ping VMs in the resource group:

    ```bash
    ansible all -m ping -i ./myazure_rm.yml
    ```

1. By default host-key checking is enabled, which may result in the following error.

    ```output
    Failed to connect to the host via ssh: Host key verification failed.
    ```

    Disable host-key verification by setting the `ANSIBLE_HOST_KEY_CHECKING` environment variable to `False`.

    ```bash
    export ANSIBLE_HOST_KEY_CHECKING=False
    ```

1. When you run the playbook, you see results similar to the following output:
  
    ```output
    ansible-inventory-test-vm1_0324 : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    ansible-inventory-test-vm2_8971 : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    ```

## Enable the VM tag

- Run the command `ansible-inventory -i myazure_rm.yml --graph` to get the following output:

    ```output
        @all:
          |--@tag_Ansible_nginx:
          |  |--ansible-inventory-test-vm1_9e2f
          |--@ungrouped:
          |  |--ansible-inventory-test-vm2_7ba9
    ```

- You can also run the following command to test connection to the Nginx VM:
  
    ```bash
    ansible -i ./myazure_rm.yml -m ping tag_Ansible_nginx
    ```

## Set up Nginx on the tagged VM

The purpose of tags is to enable the ability to quickly and easily work with subgroups of your virtual machines. For example, let's say you want to install Nginx only on virtual machines to which you've assigned a tag of `nginx`. The following steps illustrate how easy that is to accomplish:

1. Create a file named `nginx.yml`:

   ```console
   code nginx.yml
   ```

1. Paste the following sample code into the editor:

    ```yml
        ---
        - name: Install and start Nginx on an Azure virtual machine
          hosts: all
          become: yes
          tasks:
          - name: install nginx
            apt: pkg=nginx state=present
            notify:
            - start nginx
    
          handlers:
            - name: start nginx
              service: name=nginx state=started
    ```

1. Save the file and exit the editor.

1. Run the playbook using [ansible-playbook](https://docs.ansible.com/ansible/latest/cli/ansible-playbook.html)

     ```bash
     ansible-playbook  -i ./myazure_rm.yml  nginx.yml --limit=tag_Ansible_nginx
     ```

1. After running the playbook, you see output similar to the following results:

    ```output
    PLAY [Install and start Nginx on an Azure virtual machine] 

    TASK [Gathering Facts] 
    ok: [ansible-inventory-test-vm1]

    TASK [install nginx] 
    changed: [ansible-inventory-test-vm1]

    RUNNING HANDLER [start nginx] 
    ok: [ansible-inventory-test-vm1]

    PLAY RECAP 
    ansible-inventory-test-vm1 : ok=3    changed=1    unreachable=0    failed=0
    ```

## Test Nginx installation

This section illustrates one technique to test that Nginx is installed on your virtual machine.

1. Use the [az vm list-ip-addresses](/cli/azure/vm#az_vm_list_ip_addresses) command to retrieve the IP address of the `ansible-inventory-test-vm1` virtual machine. The returned value (the virtual machine's IP address) is then used as the parameter to the SSH command to connect to the virtual machine.

    ```azurecli
    ssh `az vm list-ip-addresses \
    -n ansible-inventory-test-vm1 \
    --query [0].virtualMachine.network.publicIpAddresses[0].ipAddress -o tsv`
    ```

1. While connected to the `ansible-inventory-test-vm1` virtual machine, run the [nginx -v](https://nginx.org/en/docs/switches.html) command to determine if Nginx is installed.

    ```console
    nginx -v
    ```

1. Once you run the `nginx -v` command, you see the Nginx version (second line) that indicates that Nginx is installed.

    ```output
    tom@ansible-inventory-test-vm1:~$ nginx -v

    nginx version: nginx/1.10.3 (Ubuntu)

    tom@ansible-inventory-test-vm1:~$
    ```

1. Click the `<Ctrl>D` keyboard combination to disconnect the SSH session.

1. Doing the preceding steps for the `ansible-inventory-test-vm2` virtual machine yields an informational message indicating where you can get Nginx (which implies that you don't have it installed at this point):

    ```output
    tom@ansible-inventory-test-vm2:~$ nginx -v
    The program 'nginx' can be found in the following packages:
    * nginx-core
    * nginx-extras
    * nginx-full
    * nginx-lightTry: sudo apt install <selected package>
    tom@ansible-inventory-test-vm2:~$
    ```

## Clean up resources

[!INCLUDE [ansible-delete-resource-group.md](includes/ansible-delete-resource-group.md)]

## Next steps

> [!div class="nextstepaction"]
> [Quickstart: Configure Linux virtual machines in Azure using Ansible](./vm-configure.md)
