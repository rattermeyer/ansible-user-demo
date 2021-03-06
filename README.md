
# Demo showcasing creation of users

This demo was inspired by a customer question.
In this demo, we define two default users for each new host. 
Beside the default users, there are different users per region

# How to run it
The easiest way is to run not this ansible playbooks directly, but use the
vagrant-user-demo repo instead.

# How is this organized

## The inventory
The inventory is contained in `development`

  [europe]
  192.168.33.20
  192.168.33.21
  [asia]
  192.168.33.[22:23]

This playbook defines two groups (regions) europe and asia. Both regions contain two servers.

## The playbook
 
  # site.yml
  - hosts: all
    roles:
      - users
    vars_files:
      - default_vars/default_users.yml

The playbook is applied to all defined hosts in the repository. Which users are created on each host is configured using variables.
This is the most intersting part of this play.
We make use of `group_vars`

Ansible has a mechanism that allows to define variables for each `group` in the inventory. They are placed inside the `group_vars` directory
and named according to the group. In our case the variable definition files must be called `europe` and `asia`.
Each group file defines the users that should be created especially for this region, e.g. for europe:

  users:
    - default_users
    - { name: "europe_agency", password: "xyz" }

It defines a list called `users`. This variable contains the region specific users. For europe, we only create one user `europe_agency` and say that this list should also
contain the list of `default_users` as well.

So the question is: where do the `default_users` come from? 
They are defined in the variable vile `default_users.yml` included in the site.yml

  default_users:
    - { name: "internal_user", password: "xyz" }
    - { name: "ops", password: "xyz" }

## The user role
This role creates the configured users. The main part `tasks/main.yml` is quite short

  - name: create users
    user: name={{ item.name }} password={{ item.password }} shell=/bin/bash state=present
    with_flattened: users
  - name: create .ssh directory
    authorized_key: user={{ item.name }} key="{{lookup('file', item.name +'.pub')}}"
    with_flattened: users

It handles two tasks

* Creation of new users
* setting up authorized_key for ssh login

The two tasks iterate over the `users` variable. Because we have lists in list (default_users in users), we use `with_flattened`.
One convention is used: The ssh public key is named after the user's name and stored inside the `files` directory of this role.

# Running the Example
Change to the top-level directory containing `site.yml` and call `ansible-playbook -s -i development site.yml`
The parameter `-s` is used to execute the commands with sudo.

# Check that everything worked
Run the ad-hoc command `ansible -i development -a `ls /home/` and see that each managed node contains the expected user directories.



