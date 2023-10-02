# Git-Docker-Ansible-test-build-server-automation

# Absible Playbook

########################################################################
## Creating Infrastructure
########################################################################


```
---
- name: "creating infra using ansible"
  hosts: localhost
  vars:
    region: "ap-south-1"
    access_key: "your_access_key"
    secret_key: "your_secret_key"
    project: "zomato"
    env: "prod"
  tasks:
    - name: "infra provision--creating keypair"
      amazon.aws.ec2_key:
        region: "{{ region }}"
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        name: "{{ project }}-{{ env }}"
        state: present
      register: key_status
 
      tags:
        - infra
        - build
        - test
 
    - name: "infra provisioning- copying public key as {{ project }}-{{ env }}.pem"
      when: key_status.changed == true
      copy:
        content: "{{ key_status.key.private_key }}"
        dest: "{{ project }}-{{ env }}.pem"
        mode: "0400"
 
      tags:
        - infra
        - build
        - test
 
    - name: "infra provision--creating security group"
      amazon.aws.ec2_security_group:
        name: "{{ project }}-{{ env }}-server"
        description: "{{ project }}-{{ env }}-server"
        region: "{{ region }}"
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        tags:
          "Name": "{{ project }}-{{ env }}-server"
          "project": "{{ project }}"
          "env": "{{ env }}"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0
      register: sg_server
 
      tags:
        - infra
        - build
        - test
 
    - name: "infra provision--creating ec2 instances for Build"
      amazon.aws.ec2_instance:
        region: "{{ region }}"
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        name: "{{ project }}-{{ env }}-build"
        key_name: "{{ key_status.key.name }}"
        instance_type: "t2.micro"
        exact_count: 1
        security_groups:
          - "{{ sg_server.group_id }}"
        image_id: "ami-06f621d90fa29f6d0"
        tags:
          "Name": "{{ project }}-{{ env }}-build"
          "project": "{{ project }}"
          "env": "{{ env }}"
 
      tags:
        - infra
        - build
 
    - name: "infra provision--creating ec2 instances for Test"
      amazon.aws.ec2_instance:
        region: "{{ region }}"
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        name: "{{ project }}-{{ env }}-test"
        key_name: "{{ key_status.key.name }}"
        instance_type: "t2.micro"
        exact_count: 1
        security_groups:
          - "{{ sg_server.group_id }}"
        image_id: "ami-06f621d90fa29f6d0"
        tags:
          "Name": "{{ project }}-{{ env }}-test"
          "project": "{{ project }}"
          "env": "{{ env }}"
 
      tags:
        - infra
        - test
 
 
- name: "Sleep for 60 seconds"
  hosts: localhost
  tasks:
    - name: Sleep for 60 seconds
      ansible.builtin.wait_for:
        timeout: 60
      delegate_to: localhost
 
      tags:
        - infra
        - build
 ```

########################################################################
## Dynamic inventory for Build
########################################################################

 ```
- name: "Creating dynamic inv-fetching details for build"
  hosts: localhost
  vars:
    region: "ap-south-1"
    access_key: "your_access_key"
    secret_key: "your_secret_key"
    project: "zomato"
    env: "prod"
  tasks:
    - name: "Getting Instance Details"
      amazon.aws.ec2_instance_info:
        access_key: "{{ access_key }}"
        secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          instance-state-name: ["running"]
          "tag:Name": "{{ project }}-{{ env }}-build"
          "tag:project": "{{ project }}"
          "tag:env": "{{ env }}"
      register: ec2_details
 
      tags:
        - infra
        - build
 
    - name: "Creating Dynamic Inventory for build"
      add_host:
        groups: "build"
        name: "{{ item.private_ip_address }}"
        ansible_ssh_user: "ec2-user"
        ansible_ssh_host: "{{ item.private_ip_address }}"
        ansible_ssh_port: "22"
        ansible_ssh_private_key_file: "{{ project }}-{{ env }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items: "{{ ec2_details.instances }}"
 
      tags:
        - infra
        - build
 ```
########################################################################
## Dynamic inventory for Test
########################################################################

``` 
- name: "Creating dynamic inv-fetching details for test"
  hosts: localhost
  vars:
    region: "ap-south-1"
    access_key: "your_access_key"
    secret_key: "your_secret_key"
    project: "zomato"
    env: "prod"
  tasks:
    - name: "Getting Instance Details"
      amazon.aws.ec2_instance_info:
        access_key: "{{ access_key }}"
        secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          instance-state-name: ["running"]
          "tag:Name": "{{ project }}-{{ env }}-test"
          "tag:project": "{{ project }}"
          "tag:env": "{{ env }}"
      register: ec2_details
 
      tags:
        - infra
        - test
 
 
    - name: "Creating Dynamic Inventory for test"
      add_host:
        groups: "test"
        name: "{{ item.private_ip_address }}"
        ansible_ssh_user: "ec2-user"
        ansible_ssh_host: "{{ item.private_ip_address }}"
        ansible_ssh_port: "22"
        ansible_ssh_private_key_file: "{{ project }}-{{ env }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items: "{{ ec2_details.instances }}"
 
      tags:
        - infra
        - test

 ```

########################################################################
## Provisioning - Docker Build with Image
########################################################################

``` 
- name: "Bulding Docker Image"
  become: true
  hosts: build
  vars_files:
    - packages.vars
    - docker.vars
  vars:
 
   - app_url: "https://github.com/Akshay-Gk/flaskapp-test.git"
   - clone_dir: "/var/flaskapp/"
   - image_name: "akshaygkrishnan/flask-ansible-app"
 
 
  tasks:
 
    - name: "Build - Package Installation"
      yum:
        name: "{{ packages }}"
        state: present
 
    - name: "adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - "docker"
        append: yes
 
      tags:
        - build
 
 
    - name: "Build - starting /enabling docker service"
      service:
        name: docker
        state: started
        enabled: true
 
      tags:
        - build
 
    - name: "build- installing python docker module"
      pip:
        name: "docker"
        state: absent
 
      tags:
        - build
 
    - name: "build- cloning git repo {{ app_url }}"
      git:
        repo: "{{ app_url }}"
        dest: "{{ clone_dir }}"
      register:  clone_status
 
      tags:
        - build
 
    - name: Build - Login to Docker-hub Account
      when: clone_status.changed == true
      community.docker.docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: present
 
      tags:
        - build
 
    - name: Build - "Creating Docker Image And Push To Docker-hub"
      when: clone_status.changed == true
      community.docker.docker_image:
        source: build
        build:
          path: "{{ clone_dir }}"
          pull: yes
        name: "{{ image_name }}"
        tag: "{{ item }}"
        push: true
        force_tag: yes
        force_source: yes
      with_items:
        - "{{ clone_status.after }}"
        - latest
 
      tags:
        - build
 
    - name: Build - Logout from Docker-hub Account
      when: clone_status.changed == true
      community.docker.docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: absent
 
      tags:
        - build

```

########################################################################
## Provisioning - Test Server with Container
########################################################################

``` 
- name: "running image on test server"
  hosts: test
  become: true
  vars_files:
    - testpackages.vars
  vars:
    - image_name: "akshaygkrishnan/flask-ansible-app"
 
  tasks:
    - name: Test - Installing Packages
      yum:
        name: "{{ testpackages }}"
        state: present
 
      tags:
        - test
 
    - name: Test - Adding ec2-user to docker group
      user:
        name: "ec2-user"
        groups:
          - docker
        append: yes
 
      tags:
        - test
 
    - name: Test - Installing Python Extension For Docker
      become: false
      pip:
        name: docker
      tags:
        - test
 
    - name: Test - Docker service restart/enable
      service:
        name: docker
        state: started
        enabled: true
 
      tags:
        - test
 
    - name: Test - Pulling Docker Image
      community.docker.docker_image:
        name: "{{ image_name }}"
        source: pull
        force_source: true
      register: image_status
 
      tags:
        - test
 
    - name: Test - Run Container
      when: image_status.changed == true
      community.docker.docker_container:
        name: flaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        published_ports:
          - "80:8080"
 
      tags:
        - test

```

########################################################################
## Deleting Infrastructure
########################################################################

```
- name: "Deleting Infrastructure"
  hosts: localhost
  vars:
    region: "ap-south-1"
    access_key: "your_access_key"
    secret_key: "your_secret_key"
    project: "zomato"
    env: "prod"
  tasks:
    - name: "Deleting All Instances"
      amazon.aws.ec2_instance:
        access_key: "{{ access_key }}"
        secret_key: "{{ secret_key }}"
        region: "{{ region }}"
 
        filters:
          instance-state-name: ["running"]
          "tag:Name": "{{ project }}-{{ env }}-{{ item }}"
          "tag:project": "{{ project }}"
          "tag:env": "{{ env }}"
        state: absent
      with_items:
        - build
        - test
 
      tags:
        - delete
 
    - name: "Deleting infra- sercurity group {{ project }}-{{ env }}-server"
      amazon.aws.ec2_security_group:
        access_key: "{{ access_key }}"
        secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ project }}-{{ env }}-server"
        state: absent
      tags:
        - delete
 
 
    - name: "Deleting infra- ssh key pair {{ project }}-{{ env }}"
      amazon.aws.ec2_key:
        access_key: "{{ access_key }}"
        secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ project }}-{{ env }}"
        state: absent
      tags:
        - delete

```


# Dockerfile

```
FROM alpine:latest
 
ENV FLASK_PORT 8080
 
ENV FLASK_USER flaskapp
 
ENV FLASK_ROOT_DIRECTORY /var/flaskapp/
 
RUN  mkdir $FLASK_ROOT_DIRECTORY
 
RUN adduser -h $FLASK_ROOT_DIRECTORY -s /bin/sh -D $FLASK_USER
 
WORKDIR $FLASK_ROOT_DIRECTORY
 
COPY . .
 
RUN apk update && apk add python3 py3-pip --no-cache
 
RUN pip3 install -r requirement.txt
 
RUN chown -R $FLASK_USER:$FLASK_USER $FLASK_ROOT_DIRECTORY
 
USER $FLASK_USER
 
EXPOSE  $FLASK_PORT
 
ENTRYPOINT ["python3"]
 
CMD ["app.py"]

```
