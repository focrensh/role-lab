Example 1: Deploy a Service
===========================

In this section we will create a new role and then add it to our existing playbook. The outline of the role is below.

- Create Role to deploy a BIG-IP Service

  - Create Nodes
  - Create Pool
  - Add Pool Members to Pool
  - Create Virtual Server

#. The first step will be to create the file/folder structure for the role. Luckily there is an ``ansible-galaxy init`` command that can be used to create the base directory structure for writing a role. All of the directories dont have to be utilized. The command below will create a role in the ``roles`` directory called **f5_role_service** and you should see the success message once done. From your home directory, run the following command.

   .. code:: shell
     
     ansible-galaxy init roles/f5_role_service

   .. code:: shell
     
     [student1@ansible ~]$ ansible-galaxy init roles/f5_role_service
     - roles/f5_role_service was created successfully

   With the role created you can now view the empty files/folders that were created from the command. Navigate to the role folder with ``cd roles/f5_role_service`` and look through the files.

#. We will now set the default connection details for role to use when connecting to a BIG-IP. Since these vars will be **defaults** they can easily be overwritten by specifying vars in the playbook or when calling the role. They will only be used if nothing else with higher precedence is set in the playbook or role. The connection details we are going to set, such as **private_ip** (IP of the BIG-IP) ,are defined within our environments ``inventory/host`` file.

To view the host inventory file and look at the values of the variables

   .. code:: shell
   
      cat ~/networking-workshop/lab_inventory/hosts

Now edit the ``defaults/main.yml`` file with the connection details below.

   .. code:: shell
     
     vi defaults/main.yml

   .. code:: YAML

      ---
      # defaults file for roles/f5_role_service
      provider:
        server: "{{ private_ip }}"
        user: "{{ ansible_user }}"
        password: "{{ ansible_ssh_pass }}"
        server_port: 8443
        validate_certs: no

#. The next part will be the primry part of the role. It will define the actual tasks that will run when the role is added to a playbook (Create Nodes, Pools VirtualServer, etc). Edit the ``tasks/main.yml`` file to have the tasks in the snippet below. The intent of this guide is to demonstrate roles and not modules so we will not be going in depth on the individual modules listed below. The variables defined here will be pulled from your ansible inventory file.

   .. code:: shell
     
     vi tasks/main.yml

   .. code:: YAML

      # tasks file for roles/f5_role_service
      - name: CREATE NODES
        bigip_node:
          provider: "{{provider}}"
          host: "{{hostvars[item].ansible_host}}"
          name: "{{hostvars[item].inventory_hostname}}"
        loop: "{{ groups['webservers'] }}"
      
      - name: CREATE POOL
        bigip_pool:
          provider: "{{provider}}"
          name: "http_pool"
          lb_method: "round-robin"
          monitors: "/Common/http"
          monitor_type: "and_list"
      
      - name: ADD POOL MEMBERS
        bigip_pool_member:
          provider: "{{provider}}"
          state: "present"
          name: "{{hostvars[item].inventory_hostname}}"
          host: "{{hostvars[item].ansible_host}}"
          port: "80"
          pool: "http_pool"
        loop: "{{ groups['webservers'] }}"
      
      - name: ADD VIRTUAL SERVER
        bigip_virtual_server:
          provider: "{{provider}}"
          name: "vip"
          destination: "{{private_ip}}"
          port: "443"
          enabled_vlans: "all"
          all_profiles: ['http','clientssl','oneconnect']
          pool: "http_pool"
          snat: "Automap"
      
      - name: PRINT OUT WEB VIP FOR F5
        debug:
          msg: "The VIP (Virtual IP) is https://{{ansible_host}}"

#. It is best practice to modify the ``README.md`` in the roles folder with basic information about the role. It will have a template already laid out to make filling it out easier. It is common to add a short description, examples of what variables are needed, and an example of using the role in a playbook. This is not required, but is good practice. For an idea of what to put here, looking at existing Roles on galaxy is a good place to start. The ``meta/main.yml`` allows you to also specify author, revision, and dependency information for the role as well. This information will be displayed on the Ansible Galaxy portal as well. For the sake of this guide, we can skip these steps for now.

#. Now that our Role is ready for use, lets add it to our playbook we created in the main section of this guide. Go back to your primary working directory with ``cd ~``. Open up the playbook ``role_playbook.yml`` and add the newly created role leaving the **facts** role there. It will be the same syntax as the **facts** role we added earlier.

   .. code:: YAML
   
      ---
      - name: Role Playbook
        hosts: f5
        connection: local
        gather_facts: no
      
        tasks:
      
        - include_role:
            name: focrensh.f5_role_facts
      
        - include_role:
            name: f5_role_service


#. Run the play book with ``ansible-playbook role_playbook.yml``. The playbook will return the device info as before, but it will now also create the Service defined in the new Role. You should see the new tasks run with a similar output to what is below.

   .. code:: shell

      TASK [include_role : f5_role_service] 
      
      TASK [f5_role_service : CREATE NODES] 
      changed: [f5] => (item=host1)
      changed: [f5] => (item=host2)
      
      TASK [f5_role_service : CREATE POOL] *
      changed: [f5]
      
      TASK [f5_role_service : ADD POOL MEMBERS] 
      changed: [f5] => (item=host1)
      changed: [f5] => (item=host2)
      
      TASK [f5_role_service : ADD VIRTUAL SERVER] 
      changed: [f5]
      
      TASK [f5_role_service : PRINT OUT WEB VIP FOR F5] 
      ok: [f5] =>
        msg: The VIP (Virtual IP) is https://IP
      
      PLAY RECAP 
      f5                         : ok=9    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0


   
   .. NOTE:: You should be able to now reach the F5 Service created by the role by putting the URL provided in the output in your browser. You can also log back into the BIG-IP using the same URL but with ``:8443`` at the end.

#. **Optional** As a challenge, edit the playbook so that the Service Role only runs when the Version of the BIG-IP matches what yours currently returns in the first role. This will demonstrate that the facts that the first role gathered can be used to decide future actions in your playbook!
