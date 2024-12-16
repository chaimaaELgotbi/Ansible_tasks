# Ansible tasks

## Tasks:

1. Deploy three virtual machines in the Cloud.
2. Install Ansible on one of them (control_plane).
![image](https://github.com/user-attachments/assets/f75ae780-91cf-4831-8b80-991174b2bd75)

To install ansible on the control plane (script contained in script.sh)
```
$ sudo apt update
$ sudo apt install software-properties-common
$ sudo add-apt-repository --yes --update ppa:ansible/ansible
$ sudo apt update
$ sudo apt install ansible
```

In the next step we are going to add the "pem" key pair to the Instance that has Ansible
![image](https://github.com/user-attachments/assets/d67e74b4-3053-4c68-a3ab-74fc3dd7318f)

3. Ping pong - execute the built-in ansible ping command. Ping the other two machines.
Set up a file caled inventory to group the 2 machines

```
ansible -i inventory -m ping all

```
![image](https://github.com/user-attachments/assets/33745634-402b-4d5f-8752-437148ce7fe1)

4. My First Playbook: write a playbook for installing Docker on two machines and run it.
To do this, i created a yaml file to install docker on both webservers

## Playbook contained in docker.yml
```
---
- name: Install Docker on Ubuntu
  hosts: my_ec2_instances
  become: true

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install required dependencies
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
          - python3-pip
          - virtualenv
          - python3-setuptools
        state: present
        update_cache: true

    - name: Add Docker GPG apt Key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker Repository
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
        state: present

    - name: Update apt and install docker-ce
      apt:
        name: docker-ce
        state: latest
        update_cache: true

    - name: Enable Docker service
      systemd:
        name: docker
        enabled: yes
        state: started
```

To run the playbook, used the following command:

```
ansible-playbook -i docker.yaml --syntax-check
ansible-playbook -i hosts  docker.yaml
```

![image](https://github.com/user-attachments/assets/967d3325-4336-4510-87f3-57eff64584a0)
Check Docker installation:
![image](https://github.com/user-attachments/assets/1f8a275e-8887-43f0-b6a1-767cc9f8b02e)
![image](https://github.com/user-attachments/assets/64d6881c-7683-448e-9012-febeec652c7c)


EXTRA 1. Write a playbook for installing Docker and one of the (LAMP/LEMP stack, Wordpress, ELK, MEAN - GALAXY do not use) in Docker.
Contained in wordpress.yml
```
---
- hosts: all
  become: true
  vars_prompt:
    - name: wp_db_name
      prompt: Enter the DB name

    - name: db_user
      prompt: Enter the DB username

    - name: db_password
      prompt: Enter the DB password

  vars:
    db_host: db
    wp_name: wordpress
    docker_network: wordpress_net
    # wp_host_port: "{{ lookup('env','WORDPRESS_PORT') | default(8080) }}"
    wp_container_port: 80

  tasks:
    - name: "Create a network"
      docker_network:
        name: "{{ docker_network }}"

    - name: Pull WordPress image
      docker_image:
        name: wordpress
        source: pull

    - name: Pull MySQL image
      docker_image:
        name: mysql:5.7
        source: pull

    - name: Create DB container
      docker_container:
        name: "{{ db_host }}"
        image: mysql:5.7
        state: started
        network_mode: "{{ docker_network }}"
        env:
          MYSQL_USER: "{{ db_user }}"
          MYSQL_PASSWORD: "{{ db_password }}"
          MYSQL_DATABASE: "{{ wp_db_name }}"
          MYSQL_RANDOM_ROOT_PASSWORD: '1'
        volumes:
          - db:/var/lib/mysql:rw
        restart_policy: always

    - name: Create WordPress container
      docker_container:
        name: "{{ wp_name }}"
        image: wordpress:latest
        state: started
        ports:
          - "80:80"
        restart_policy: always
        network_mode: "{{ docker_network }}"
        env:
          WORDPRESS_DB_HOST: "{{ db_host }}:3306"
          WORDPRESS_DB_USER: "{{ db_user }}"
          WORDPRESS_DB_PASSWORD: "{{ db_password }}"
          WORDPRESS_DB_NAME: "{{ wp_db_name }}"
        volumes:
          - wordpress:/var/www/html
```


EXTRA 2. Playbooks should not have default creds to databases and/or admin panel.
![image](https://github.com/user-attachments/assets/940c8cd9-14e1-44f7-99c4-71dcaf6ff0e2)

EXTRA 3. For the execution of playbooks, dynamic inventory must be used (GALAXY can be used).
For this task, since the ansible documentation recommends pluggins for dynamic inventories over scripts, i will be using aws pluggins for this
1. Create an IAM user to get a secret access id and secret access key.
2. Next, create an IAM role and gave it ec2 access, then attache the role to control plane
3. Create a new file called __aws_ec2.yml

```
---
plugin: aws_ec2
aws_access_key: <access_id>
aws_secret_key: <access_key>
keyed_groups:
  - key: tags
    prefix: tag
  - prefix: instance_type
  - key: placement.region
    prefix: aws_region
```
4. Run the following command on it:
```
ansible-inventory -i <Path to file> --list
```
![image](https://github.com/user-attachments/assets/9d2f5fa6-601e-4d6c-b4c7-0d8a8ae52e47)

```
ansible all -m ping
```
![image](https://github.com/user-attachments/assets/d57b8d81-dfbd-44ed-8a7e-52404a4e1133)

```
ansible-inventory --graph
```
![image](https://github.com/user-attachments/assets/f580f882-db60-4880-a0bf-91ab7e6b8817)





