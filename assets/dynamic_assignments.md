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
In the same version, variants of import were also introduces, such as:

import_role
import_tasks
We made use of a special variables {{ playbook_dir }} and {{ inventory_file }}. {{ playbook_dir }} will help Ansible to determine the location of the running playbook, and from there navigate to other path on the filesystem. {{ inventory_file }} on the other hand will dynamically resolve to the name of the inventory file being used, then append .yml so that it picks up the required file within the env-vars folder.
We are including the variables using a loop. with_first_found implies that, looping through the list of files, the first one found is used. This is good so that we can always set default values in case an environment specific env file does not exist.