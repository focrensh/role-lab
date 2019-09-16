Example 1: Deploy a Service
=========================================

In this section we will create a new role and then add it to our existing playbook. The outline of the role is below.

- Create Role to deploy BIG-IP Service
  - Create Nodes
  - Create Pool
  - Add Pool Members to Pool
  - Create Virtual Server

#. The first step will be to create the file/folder structure for the role. Luckily there is an ``ansible-galaxy init`` command that can be used to create the base directory structure for writing a role. All of the directories dont have to be utilized. The command below will create a role in the ``roles`` directory called **f5_role_service** and you should see the success message once done.

   .. code:: shell
     
     [student1@ansible networking-workshop]$ ansible-galaxy init roles/f5_role_service
     - roles/f5_role_service was created successfully

   With the role created you can now view the empty files/folders that were created from the role. Navigate to the role folder with ``cd roles/f5_role_service`` and look through the files.

#. Next we will set the default connection details for role to use when connecting to a device. Since these vars will be **defaults** they can easily be overwritten by specifying vars in the playbook or when calling the role. They will only be used if nothing else with higher precdence is set. Edit the ``defaults/main.yml`` file with the connection details below.

   .. code:: YAML

      ---
      # defaults file for roles/f5_role_service
      provider:
        server: "{{private_ip}}"
        user: "{{ansible_user}}"
        password: "{{ansible_ssh_pass}}"
        server_port: 8443
        validate_certs: no

#. The next part will be the primry part of the role. It will define the actual tasks that will run when the role is added to a playbook. Edit the ``tasks/main.yml`` file to have the tasks in the snippet below. The intent of this guide is to demonstrate roles and not modules so we will not be going in depth on the individual modules listed below. The variables defined here will be pulled from your ansible inventory file.

   .. code:: YAML

      ---
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

#. For good practice you should now modify the ``README.md`` in the roles folder with basic information about the role. It will have a template already laid out to make filling it out easier. It is common to add a short description, examples of what variables are needed, and an example of using the role in a playbook. This is not required, but is good practice. For an idea of what to put here, looking at existing Roles on galaxy is a good place to start.

#. Now that our Role is ready for use, lets add it to our playbook we created in the main section of this guide. Go back to your primary working directory with ``cd /home/student1/networking-workshop/``. Open up the playbook ``role_playbook.yml`` and add the newly created role leaving the **facts** role there. It will be the same syntax as the **facts** role we added earlier.

#. Run the play book with ``ansible-playbook role_playbook.yml``. The playbook will return the device info as before, but it will now also create the Service defined in the new Role.

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


#. **Optional** As a challenge, edit the playbook so that the Service Role only runs when the Version of the BIG-IP matches what yours currently returns in the first role.