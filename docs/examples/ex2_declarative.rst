Example 2: Declarative APIs
===========================

In this section we will create a new role that deploys the same service but using F5s AS3 (|as3link|) interface.  AS3 uses a declarative model, meaning you provide a JSON declaration rather than a set of imperative commands or modules. The declaration represents the configuration which AS3 is responsible for creating on a BIG-IP system. AS3 is an application-centric schema for deploying Layer 4-7 Application Services on BIG-IP devices. Benefits of AS3 include:

- |atomic| (all or nothing)
- Built in |idempotent|
- Declarative (state desired state, vs each explicit steps)


In this section you will..

- Create a Role to deploy a Service using AS3

  - F5 module bigip_wait which verifies the API is up and ready
  - URI module API Call sending desired Service declaration

    - Add Jinja2 template for the AS3 declaration to ``templates`` directory of the role
  - Debug statement to print the results of the API call
  
- Replace the previous service role in our playbook with the new one created

.. attention:: This role will create similar objects to the last role, but using AS3 instead. You will need to delete the objects created by       the last role in order to avoid conflicts. Run the following playbook to cleanup your BIG-IP. ``ansible-playbook ~/networking-workshop/2.1-delete-configuration/bigip-delete-configuration.yml``

#. The first step will be to create the file/folder structure for the role. Similar to the last example, we will use the ``ansible-galaxy init`` command to create the base directory structure for the role. From your home directory (``cd ~``), run the following command.

   .. code:: shell
     
     cd ~
     ansible-galaxy init roles/f5_role_as3

   .. code:: shell
     
     [student1@ansible ~]$ ansible-galaxy init roles/f5_role_as3
     - roles/f5_role_as3 was created successfully

   With the role created you can now view the empty files/folders that were created from the command. Navigate to the role folder with ``cd roles/f5_role_as3``.

#. Just like the last role, we will set the default connection details for role to use when connecting to a BIG-IP. Since these vars will be **defaults** they can easily be overwritten by specifying vars in the playbook or when calling the role. They will only be used if nothing else with higher precedence is set in the playbook or role. The connection details we are going to set, such as **private_ip** (IP of the BIG-IP) ,are defined within our environments ``inventory/host`` file.

   To view the host inventory file and look at the values of the variables

   .. code:: shell
   
      cat ~/networking-workshop/lab_inventory/hosts

   Now edit the ``defaults/main.yml`` file with the connection details below. 

   .. code:: shell
     
     vi defaults/main.yml

   .. code:: YAML

      ---
      # defaults file for roles/f5_role_as3
      provider:
        server: "{{ private_ip }}"
        user: "{{ ansible_user }}"
        password: "{{ ansible_ssh_pass }}"
        server_port: 8443
        validate_certs: no

#. Now we will update the tasks for the role under ``tasks/main.yml``. In the snippet below you will see 3 tasks. The first task , **bigip_wait**, verifies that the remote BIG-IP API is ready for requests. The 2nd task, **uri**, makes a POST API call to the AS3 endpoint on the BIG-IP. The payload of the API call is a **Jinja2** template which we will define and review later. The final task , **debug**, displays the status of the API call. Edit the ``tasks/main.yml`` file to have the tasks in the snippet below. Again, the variables defined here will be pulled from your ansible inventory file.

   .. code:: shell
     
      vi tasks/main.yml

   .. code:: YAML

      ---
      # tasks file for roles/f5_role_as3
      - name: Wait for API to be up
        bigip_wait:
          timeout: 150
          provider: "{{ provider }}"
        delegate_to: localhost
      
      - name: Push AS3 Declaration
        uri:
          url: "https://{{ provider.server }}:{{ provider.server_port }}/mgmt/shared/appsvcs/declare"
          method: POST
          user: "{{ provider.user }}"
          password: "{{ provider.password }}"
          body: "{{ lookup('template', 'as3.j2') }}"
          status_code: 200
          timeout: 30
          body_format: json
          validate_certs: no
        register: as3_task
        delegate_to: localhost
      
      - debug: var=as3_task.json.results


   Above in the 2nd task, you see that the body of the API call will be templated from a file called **as3.j2**. Take note that we are calling this file as a **template** which tells ansible to replace variables found within it. In the next step we will define this file.

#. Something new for this role will be the use of **templates** and **Jinja2**. If you are not familiar with templating, you can read more about |j2|. We will be using it to replace objects defined within a JSON payload with varaibles from our **inventory-hosts** file. Create a new file in the ``templates/`` directory called ``as3.j2``. Run the command below and copy the snippet into the file.

   .. code:: shell
     
      vi templates/as3.j2

   .. code:: shell
     
      {
          "class": "AS3",
          "action": "deploy",
          "persist": true,
          "declaration": {
              "class": "ADC",
              "schemaVersion": "3.2.0",
              "id": "testid",
              "label": "test-label",
              "remark": "test-remark",
              "WorkshopExample":{
                  "class": "Tenant",
                  "web_app": {
                      "class": "Application",
                      "template": "http",
                      "serviceMain": {
                          "class": "Service_HTTP",
                          "virtualAddresses": [
                              "{{ private_ip }}"
                          ],
                          "pool": "app_pool"
                      },
                      "app_pool": {
                          "class": "Pool",
                          "monitors": [
                              "http"
                          ],
                          "members": [
                              {
                                  "servicePort": 80,
                                  "serverAddresses": [
                                      {% set comma = joiner(",") %}
                                      {% for mem in groups['webservers'] %}
                                          {{comma()}} "{{  hostvars[mem]['ansible_host']  }}"
                                      {% endfor %}
                                  ]
                              }
                          ]
                      }
                  }
              }
          }
      }


   If you look within the Jinja2 template above, you can see that there are a few variables defined (items with '{{  }}' around them') which will be replaced by the role when it is run. The **private_ip** will go in place for the VirualAddress ( ie `{{ private_ip }}` ) of the service and the pool members of the service will be created from iterating over the **webservers** group in our inventory ( ie `{% for mem in groups['webservers'] %}` ). This is only an example template and could have variables which best fit your production environment.
   
   One other thing to note here is that the task refers to the template by only its name of **as3.j2** and not its full path. The role knows to look for templates in the ``templates`` directory, so the full path is not needed. This is some of the "for-free" logic you get by following ansible-galaxy predefined folder structure.

#. As mentioned on the previous role, it is best practice to update the ``README.md`` and ``meta/main.yml`` with information about the roles intent and usage. We will skip this again for brevity.

#. Now that our Role is ready for use, lets replace the last role we added to our playbook with the new one using AS3. Go back to your primary working directory with ``cd ~``. Open up the playbook ``role_playbook.yml`` and modify the 2nd role included to be ``f5_role_as3`` as below. Make sure you are back in your home directory with ``cd ~``.

   .. code:: shell
     
      vi role_playbook.yml

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
            name: f5_role_as3

        - name: PRINT OUT WEB VIP FOR F5
          debug:
            msg: "The VIP (Virtual IP) is https://{{ansible_host}}"

#. Run the play book with ``ansible-playbook role_playbook.yml``. The playbook will once again return the device facts as before, but it will now create the Service defined in the new Role using AS3. You should see the new tasks run with a similar output to what is below.

   .. code:: shell

      TASK [include_role : f5_role_as3]
      
      TASK [f5_role_as3 : Wait for API to be up] 
      ok: [f5 -> localhost]
      
      TASK [f5_role_as3 : Push AS3 Declaration] 
      ok: [f5 -> localhost]
      
      TASK [f5_role_as3 : debug] 
      ok: [f5] =>
        as3_task.json.results:
        - code: 200
          host: localhost
          lineCount: 19
          message: success
          runTime: 2932
          tenant: WorkshopExample
      
      PLAY RECAP 
      f5                         : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0


   .. NOTE:: You should be able to now reach the F5 Service created by the role by putting the URL provided in the output in your browser. You can also log back into the BIG-IP using the same URL but with ``:8443`` at the end.

   The power of using declarative tools such as **AS3** comes that you now only have to manage the single API to provision your entire service. By abstracting the imperative complexity of this task away, it allows you to focus your time on adding further integration into your playbooks and environment. If the `results` output in the call above failed, then you would not have to worry about backing any of the configuration out since the entire service is **Atomic** as mentioned before (Its all or nothing!).


.. |as3link| raw:: html

   <a href="https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/" target="_blank">Application Services 3 Extension</a>
.. |atomic| raw:: html

   <a href="https://www.techopedia.com/definition/3466/atomic-operation" target="_blank">Atomic</a>
.. |idempotent| raw:: html

   <a href="https://whatis.techtarget.com/definition/idempotence" target="_blank">Idempotency</a>
.. |j2| raw:: html

   <a href="https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html" target="_blank">Jinja2 here</a>


   