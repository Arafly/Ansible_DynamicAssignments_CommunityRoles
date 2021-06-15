## Ansible Dynamic Assignments (Include) and Community Roles

In this project we'll introduce dynamic assignments by using include module.

In Project 12, you can already tell that static assignments use import Ansible module. The module that enables dynamic assignments is include.

Hence,

```
import = Static
include = Dynamic
```

When the **import** module is used, all statements are **pre-processed** at the time playbooks are parsed. Meaning, when you execute site.yml playbook, Ansible will process all the playbooks referenced during the time it is parsing the statements. 

On the other hand, when **include** module is used, all statements are **processed only during** execution of the playbook. Meaning, after the statements are parsed, any changes to the statements encountered during execution will be used.


### Introducing Dynamic Assignment Into Our structure

- Create a new branch in your https://github.com/<your-name>/automate-everything repository  and name it *dynamic-assignments*.

- Also create a new folder in your 'ansible-artifact' directory, name it dynamic-assignments. Then inside this folder, create a new file and name it *env-vars.yml*. We'll instruct site.yml to include this playbook later.

Since we'll be using the same Ansible to configure multiple environments, and each of these environments will have certain unique attributes, such as servername, ip-address etc., we will need a way to set values to variables per specific environment.

For this reason, we will now create a folder to keep each environment’s variables file. Therefore, create a new folder 'env-vars', then for each environment, create new YAML files which we will use to set variables.

Your layout should now look like this.

```
├── dynamic-assignments
│   └── env-vars.yml
├── env-vars
    └── dev.yml
    └── stage.yml
    └── uat.yml
    └── prod.yml
├── inventory
    └── dev
    └── stage
    └── uat
    └── prod
├── playbooks
    └── site.yml
└── static-assignments
    └── common.yml
    └── webservers.yml
```

Now paste the instruction below into the env-vars.yml file.

```
---
- name: collate variables from env specific file, if it exists
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ playbook_dir }}/../env_vars/{{ "{{ inventory_file }}.yml"
    - "{{ playbook_dir }}/../env_vars/default.yml"
  tags:
    - always

- name: Webserver assignment
  hosts: webservers
    - import_playbook: ../static-assignments/webservers.yml
```
Observe 3 things to notice here:

We used include_vars syntax instead of include, this is because Ansible developers decided to separate different features of the module. From Ansible version 2.8, the include module is deprecated and variants of include_* must be used. These are:
include_role
include_tasks
include_vars

We made use of a special variables `{{ playbook_dir }}` and `{{ inventory_file }}`. The `{{ playbook_dir }}` will help Ansible to determine the location of the running playbook, and from there navigate to other path on the filesystem. While `{{ inventory_file }}` on the other hand will dynamically resolve to the name of the inventory file being used, then append .yml so that it picks up the required file within the env-vars folder.
We are including the variables using a loop. `with_first_found` implies that, looping through the list of files, the first one found is used. This is good so that we can always set default values in case an environment specific env file does not exist.

### Update site.yml with dynamic assignments

Update the *site.yml* file to make use of the dynamic assignment. (At this point, we cannot test it yet. We are just setting the stage for what is yet to come)

site.yml should now look like this.

```
---
- name: Include dynamic variables 
  hosts: all
  tasks:
    - import_playbook: ../static-assignments/common.yml 
    - include_playbook: ../dynamic-assignments/env-vars.yml
  tags:
    - always

- name: Webserver assignment
  hosts: webservers
    - import_playbook: ../static-assignments/webservers.yml
```

### Community Roles
Now it is time to create a role for MySQL database - it should install the MySQL package, create a database and configure users. But why should we re-invent the wheel? There are tons of roles that have already been developed by other open source engineers out there. These roles are actually production ready, and dynamic to accomodate most of Linux flavours. With Ansible Galaxy again, we can simply download a ready to use ansible role, and keep going.

- Download Mysql Ansible Role
You can browse available community roles here

We will be using a MySQL role developed by geerlingguy.

Hint: To preserve your your GitHub in actual state after you install a new role - make a commit and push to master your ‘ansible-config-mgt’ directory. Of course you must have git installed and configured on Jenkins-Ansible server and, for more convenient work with codes, you can configure Visual Studio Code to work with this directory. In this case, you will no longer need webhook and Jenkins jobs to update your codes on Jenkins-Ansible server, so you can disable it - we will be using Jenkins later for a better purpose.

On Jenkins-Ansible server make sure that git is installed with git --version, then go to ‘ansible-config-mgt’ directory and run

```
git init
git pull https://github.com/<your-name>/ansible-config-mgt.git
git remote add origin https://github.com/<your-name>/ansible-config-mgt.git

# config
git config --global user.email "you@example.com"
git config --global user.name "Your Name"

# A fix to this error message
# error: The following untracked working tree files would be overwritten by merge...

git commit -m "your message"
git pull origin master --allow-unrelated-histories
git add -A .
git stash
sudo git pull origin master --allow-unrelated-histories

# merge the conflicts
sudo git add .

git push --set-upstream origin master

git checkout -b roles-feature
git branch -vv
```

Inside roles directory create your new MySQL role with ansible-galaxy install geerlingguy.mysql and rename the folder to mysql

`mv geerlingguy.mysql/ mysql`


Read README.md file, and edit roles configuration to use correct credentials for MySQL required for the tooling website.

Now it is time to upload the changes into your GitHub:

```
git add .
git commit -m "Commit new role files into GitHub"
git push --set-upstream origin roles-feature
```

Now, if you are satisfied with your codes, you can create a Pull Request and merge it to main branch on GitHub.

### Role for Load Balancer
We want to be able to choose which Load Balancer to use, Nginx or Apache, so we need to have two roles respectively:

- Nginx
- Apache

- Decide if you want to develop your own roles (apache), or find available ones from the community(nginx).

> For the apache role. I cloned the work of Shubham Rasal at this <https://github.com/ShubhamRasal/ansible-playbooks.git>, as I couldn't find a exclusive apache role on the ansible-galaxy.

- Check afterwards by running:

`$ ansible-galaxy role list`

```
Output
- myapache, (unknown version)
- webserver, (unknown version)
- mysql, (unknown version)
- nginx, 0.20.0
```

- Update both static-assignment and site.yml files to refer the roles
Important Hints:

> Since you cannot use both Nginx and Apache load balancer, you need to add a condition to enable either one - this is where you can make use of variables.

- Declare a variable in defaults/main.yml file inside the Nginx and Apache roles. Name each variables `enable_nginx_lb` and `enable_apache_lb` respectively.
- Set both values to false like this:
   enable_nginx_lb: false
   enable_apache_lb: false.
- Declare another variable in both roles load_balancer_is_required and set its value to false as well
Update both assignment and site.yml files respectively
loadbalancers.yml file


```
- hosts: lb
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
site.yml file

     - name: Loadbalancers assignment
       hosts: lb
         - import_playbook: ../static-assignments/loadbalancers.yml
        when: load_balancer_is_required 
```

Now you can make use of env-vars\uat.yml file to define which loadbalancer to use in UAT environment by setting respective environmental variable to true. We'll activate load balancer, and enable nginx by setting these in the respective environment’s env-vars file.

```
enable_nginx_lb: true
load_balancer_is_required: true
```

The same must work with apache LB, so we can switch it by setting respective environmental variable to true and other to false.

To test this, you can update inventory for each environment and run Ansible against each environment.

`$ ansible-playbook -i /home/araflyayinde/ansible-artifact/inventory/dev /home/araflyayinde/ansible-artifact/playbooks/site.yml`

```
Output

[DEPRECATION WARNING]: The TRANSFORM_INVALID_GROUP_CHARS settings is set to allow bad characters in group names by default, this will change, but still be user 
configurable on deprecation. This feature will be removed in version 2.10. Deprecation warnings can be disabled by setting deprecation_warnings=False in 
ansible.cfg.
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
PLAY [Ensure to include dynamic variables] ************************************************************************************************************************
TASK [Gathering Facts] ********************************************************************************************************************************************
ok: [mysql]
ok: [file-storage]
ok: [webserver1]
ok: [webserver2]
ok: [nginx]
TASK [collate variables from env specific file, if it exists] *****************************************************************************************************
fatal: [file-storage]: FAILED! => {"msg": "No file was found when using first_found. Use errors='ignore' to allow this task to be skipped if no files are found"}
fatal: [webserver2]: FAILED! => {"msg": "No file was found when using first_found. Use errors='ignore' to allow this task to be skipped if no files are found"}
fatal: [webserver1]: FAILED! => {"msg": "No file was found when using first_found. Use errors='ignore' to allow this task to be skipped if no files are found"}
fatal: [mysql]: FAILED! => {"msg": "No file was found when using first_found. Use errors='ignore' to allow this task to be skipped if no files are found"}
fatal: [nginx]: FAILED! => {"msg": "No file was found when using first_found. Use errors='ignore' to allow this task to be skipped if no files are found"}
PLAY RECAP ********************************************************************************************************************************************************
file-storage               : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
mysql                      : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
nginx                      : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
webserver1                 : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
webserver2                 : ok=1    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
```

Please ignore the failure. The point is to be able to activate the load balancer, to dis/enable nginx or apache by setting these in the respective environment’s env-vars file.

### Congratulations!
