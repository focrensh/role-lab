Build a Role Lab
================

.. contents:: :depth: 3

.. toctree::
   :maxdepth: 2
   :caption: Examples:

   examples/ex1_modules
   examples/ex2_declarative


What is an Ansible Role
-----------------------

Ansible role is an independent component which allows reuse of common configuration steps. Ansible roles have to be used within playbook. An Ansible role is a set of tasks to configure a device to a desired state. Roles are defined using YAML files with a predefined directory structure. This predefined structure allows a roles behavior to be predictable when reusing them.

We will go over the directory structure later in the lab

For complete documentation refer to https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html

Why use an Ansible Role
-----------------------

Advantages: The proper organization of roles will not only simplify your work with scripts, improving their structure and further support, but also eliminates duplication of tasks in playbooks. A role is a way of splitting Ansible tasks into files which are easier to manipulate with.

Role Structure
--------------

Below is the starting structure of a role. A brief description of each directory/files purpose is listed below.

.. code:: shell

   role-name
     ├── README.md
     ├── defaults
     │   └── main.yml
     ├── files
     ├── handlers
     │   └── main.yml
     ├── meta
     │   └── main.yml
     ├── tasks
     │   └── main.yml
     ├── templates
     ├── tests
     │   ├── inventory
     │   └── test.yml
     └── vars
         └── main.yml

- Most directorys will have a ``main.yml``. The role will check this file by default for instructions.
- The tasks directory is the core of the role. It is the tasks or actions that the role will take. Typically this contains modules.
- handlers - contains handlers, which may be used by this role or even anywhere outside this role.
- defaults - default variables for the role (see Using Variables for more information).
- vars - other variables for the role (see Using Variables for more information). These take precedence of **defaults**.
- files - contains files which can be deployed via this role.
- templates - contains templates which can be deployed via this role. (inlcuding jinja2)
- tests - good for automated testing of role, but will not be covered today
- README (markdown) is important for adding descriptions and context to your role for others to use
- meta - contains information that ansible-galaxy uses to define the role (Author, dependencies, etc)

This structure may seem overwhelming but there is a helpful command we will cover later for getting started which will create all of these folders/files for you.

What is Ansible Galaxy
----------------------

Ansible Galaxy refers to the Galaxy portal/hub where users can share roles, and to a command line tool for installing, creating, and managing roles. It it open to the community, anyone can publish and consume roles form galaxy.

Example roles in Galaxy
~~~~~~~~~~~~~~~~~~~~~~~

- Installing NGINX regardless of destination platform. (https://galaxy.ansible.com/nginxinc/nginx)

  - The Role can figure out if its debian or ubuntu or others and run the appropriate modules/commands to install nginx

- Group common tasks together to get a desired outcome

  - Role to deploy a global load balancing solution (Create WIDE-IP, POOL, Servers, etc): |f5gslbrole|
  - Role to troublshoot the BIG-IP (create tech support, backup and restore confguration etc): |f5backuprole|
  
Complete list of |f5ansibleroles| in galaxy

Installing Roles from Galaxy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default **ansible** will look for and install roles in 1 of 2 places:

   - Releative to current playbook being run: ``./roles`` 
   - Global for the ansible install: ``/etc/ansible/roles``

#. Navigate to the ``networking-workshop/`` directory.

   .. code:: shell

      [student1@ansible ~]$ cd networking-workshop/
      [student1@ansible networking-workshop]$

#. Now we are going to create a folder relative to our current working directory to store our ansible roles.

   .. code:: shell

      [student1@ansible networking-workshop]$ mkdir roles

#. Next we will update the **ansible.cfg** file for our current user to tell ansible to find and install roles in the new folder we created. At the bottom of the file we will add the line ``roles_path = ./roles``.

   .. code:: shell

      [student1@ansible networking-workshop]$ vi ~/.ansible.cfg
      [defaults]
      connection = smart
      timeout = 60
      inventory = /home/student1/networking-workshop/lab_inventory/hosts
      host_key_checking = False
      private_key_file = /home/student1/.ssh/aws-private.pem
      roles_path = ./roles

#. Lets view what current roles are installed (it should be none). In the current terminal run ``ansible-galaxy list``.

   .. code:: shell

      [student1@ansible networking-workshop]$ ansible-galaxy list
      # /---/.ansible/roles

   For the current environment, it will list out the path for each defined location roles are stored. If there are any roles, they will show here. Once we install and create roles later on, you can run this command again to see them.

#. Now that our environment is ready to use roles, lets install one from **Ansible Galaxy**. Navigate to |f5rolefacts| and install the role. Spend some time looking at the **Read Me** page for the role as well as this is generated from the README file in the role structure. The **Details** page of this role will provide an installation snippet like below. Copy this command using the **copy** icon and paste it into your terminal. The Role should install and the output should look similar to the shell output below.

   |role-install|

   Output:

   .. code:: shell

      [student1@ansible networking-workshop]$ ansible-galaxy install focrensh.f5_role_facts
        - downloading role 'f5_role_facts', owned by focrensh
        - downloading role from https://github.com/focrensh/f5-role-facts/archive/master.tar.gz
        - extracting focrensh.f5_role_facts to /home/student1/networking-workshop/roles/focrensh.f5_role_facts
        - focrensh.f5_role_facts (master) was installed successfully

#. Run the ``ansible-galaxy list`` command again to see that the new role is installed.

#. Spend some time looking through the role folders/files in ``./roles/focrensh/f5_role_facts``. You will see many of the files mentioned before. The primary file to look at would be the ``tasks/main.yml`` as it has the tasks that will take place when the role is called. You will see tasks in this file for gathering facts from the BIG-IP and then parsing the fact to return information about the BIG-IP such as MAC and VERSION. These should look the same as tasks that you would normally just put inside a playbook directly.

Referencing a Role in a playbook
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Now that we have downloaded a role, lets look how to add it into an Ansible playbook. There is more than one way to reference a role within a playbook.

Classic (original way) - ansible will check each roles directory for tasks/handlers/vars/default vars and other objects to add for the current host within the playbook.

.. code:: rst

   ---
   - hosts: f5
     roles:
      - super_awesome_role
      - betterer_role

Use Roles inline (2.4+)

.. code:: yaml

   ---
   - hosts: f5
     tasks:
     - import_role:
         name: super_awesome_role
     - include_role:
         name: betterer_role
       
- Import (static) vs Include (dynamic)

  - Import tasks are treated more like part of the actual playbook.
  - Include tasks are added when the playbook gets to those tasks.
  - Include can loop since it’s a tasks
  - Include cannot reference/view objects within tasks such as (--list-tasks , --start-at-task, etc)

Roles can use vars, tags, and conditionals just like other tasks. Below is an example of adding 2 variables to a role inline of the playbook.

   .. code:: YAML
   
      - hosts: f5
        tasks:
        - include_role:
           name: super_awesome_role
          vars:
           dir: '/opt/test'
           app_port: 8080


#. Lets create a playbook that will call the role we created in the previous section. Create a new playbook called ``role_playbook.yml`` and copy the yaml below into it. The name of the role matches that of the role that is now inside of the ``./roles`` directory. Notice that we are including the role into the playbook similar to tasks. When the ``include_role`` task triggers it will run the tasks within it as if they were then part of the playbook.

   .. code:: yaml

      ---
      - name: Role Playbook
        hosts: f5
        connection: local
        gather_facts: no
      
        tasks:
      
        - include_role:
            name: focrensh.f5_role_facts

#. Run the play book with ``ansible-playbook role_playbook.yml``. The playbook>>role should output the VirtualServers, MAC, and VERSION of the BIG-IP.

   .. code:: shell
   
      TASK [focrensh.f5_role_facts : DISPLAY THE MAC ADDRESS]
      ok: [f5] => {
          "device_facts['system_info']['base_mac_address']": "02:d1:fc:72:ba:aa"
      }
      TASK [focrensh.f5_role_facts : DISPLAY THE VERSION] 
      ok: [f5] => {
          "device_facts['system_info']['product_version']": "14.1.0.3"
      }


   The example facts that come back from this role could now be used to take action in other parts of the playbook. We have access to the variables without having to define each task to gather/parse them directly in the playbook. We can simply include the role into the playbook making it less difficult to understand the intent of the playbook and also allowing other playbooks to reuse this role.


Creating Roles
--------------

Role1 - Using Ansible modules
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* :doc:`examples/ex1_modules`

Now that the role is ready let's take a look at how we can reference a role within a playbook for execution

Role2 - Using AS3 
~~~~~~~~~~~~~~~~~

* :doc:`examples/ex2_declarative`

Build your own F5 Ansible Role
------------------------------

Above are examples of how to develop a role and what a role with F5 BIG-IP would look like. Below are a few more use-cases for which a role can be developed. 

You are encouraged to pick one of the use cases below and or come up with your own F5 BIG-IP use case and build a role for it. If completed we will upload the role to Ansible Galaxy for the community to be able to consume.

- Upload and attach iRules
- Display relevant information about BIG-IP (software version/hardware etc.)
- Parse virtual server information and display the default pool hence and pool members that belong to the pool

Take a look at |f5ansiblemodules| available and get started 

Upload a role to Galaxy
-----------------------
Upload role to galaxy DEMO


.. |role-install| image:: images/galaxy-install.png
   :scale: 100%

.. |f5ansibleroles| raw:: html

   <a href="https://galaxy.ansible.com/f5devcentral" target="_blank">F5 Ansible Roles</a>

.. |f5backuprole| raw:: html

   <a href="https://galaxy.ansible.com/f5devcentral/backup_config" target="_blank">F5 Backup</a>

.. |f5gslbrole| raw:: html

   <a href="https://galaxy.ansible.com/f5devcentral/bigip_gslb" target="_blank">F5 GSLB</a> 

.. |f5rolefacts| raw:: html

   <a href="https://galaxy.ansible.com/focrensh/f5_role_facts" target="_blank">F5 Facts</a> 

.. |f5ansiblemodules| raw:: html

   <a href="https://docs.ansible.com/ansible/latest/modules/list_of_network_modules.html#f5" target="_blank">F5 Ansible Modules</a> 
