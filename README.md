# ansible-beginner-project


### Ansible basic project workflow

- Create an Ubuntu EC2 instance on AWS - save the key-pair in a known directory locally

    - This is the managed node (host node)

- SSH into this instance to download dependencies

```zsh
$ ssh -i /path/to/your/file.pem ubuntu@ec2-XX-XX-XXX-XXX.eu-west-2.compute.amazonaws.com
```

- Install ansible

```zsh
$ sudo apt-get update

$ sudo apt-add repository -y ppa:ansible/ansible

$ sudo apt-get update

$ sudo apt-get install -y ansible

$ ansible --version
```

- Install boto framework - Ansible will access AWS resources using boto SDK
```zsh
$ sudo pip3 install boto boto3

$ pip3 list boto | grep boto # this will show you the version of boto installed
```

- Provisioning target EC2 instance using Ansible managed host (EC2 instance in step 1)
    - Pre-requisite: Assign AmazonEC2FullAccess role to the Ansible managed host - you can do this via the AWS console
        - Click on managed host instance > Actions > Security > Modify IAM role > Select correct role > Save > Exit and SSH back into instance
- Commands to run in managed host
```zsh
$ sudo vi /etc/ansible/hosts

# Go to the bottom of the file and add the following group entry

[localhost]
local

# ESC and :wq! to exit vim

$ cd ~

$ mkdir playbooks && cd playbooks

$ sudo vi create_ec2.yaml
```

- Use the following playbook and changed necessary parts (highlighted):

```yml
---
 - name: Provisioning EC2 instances using Ansible
   hosts: localhost
   connection: local
   gather_facts: False
   tags: provisioning

   vars:
     keypair: MyEC2Key                            # change
     instance_type: t2.small
     image: ami-020db2c14939a8efb                 # change
     wait: yes
     group: webserver
     count: 1
     region: us-east-2                            # change (optional)
     security_group: my-security-grp
   
   tasks:

     - name: Task # 1 - Create my security group
       local_action: 
         module: ec2_group
         name: "{{ security_group }}"
         description: Security Group for webserver Servers
         region: "{{ region }}"
         rules:
            - proto: tcp
              from_port: 22
              to_port: 22
              cidr_ip: 0.0.0.0/0
            - proto: tcp
              from_port: 8080
              to_port: 8080
              cidr_ip: 0.0.0.0/0
            - proto: tcp
              from_port: 80
              to_port: 80
              cidr_ip: 0.0.0.0/0
         rules_egress:
            - proto: all
              cidr_ip: 0.0.0.0/0
       register: basic_firewall
     - name: Task # 2 Launch the new EC2 Instance
       local_action:  ec2 
                      group={{ security_group }} 
                      instance_type={{ instance_type}} 
                      image={{ image }} 
                      wait=true 
                      region={{ region }} 
                      keypair={{ keypair }}
                      count={{count}}
       register: ec2
     - name: Task # 3 Add Tagging to EC2 instance
       local_action: ec2_tag resource={{ item.id }} region={{ region }} state=present
       with_items: "{{ ec2.instances }}"
       args:
         tags:
           Name: Ansible-target-server
```

- Save file and run playbook
```zsh
$ ansible-playbook create_ec2.yaml
```

- If all goes well, your EC2 instance should appear on the AWS console in your chosen region

- Creating the connection between ansible management instance and the target instance
    - Generate SSH keys
    ```zsh
    $ ssh-keygen
    ```
    - Copy the public key information
    ```zsh
    $ sudo cat ~/.ssh/id_rsa.pub
    ```
    - Target Instance: Now SSH into the target instance you created earlier and paste the copied content into the authorised key file in the .ssh directory
    - DON’T delete existing key - Shift a will take you to the end of the line, press ENTER once, and paste clipboard content
    - Host instance: SSH into the new instance to test the connection, then exit
    ```zsh
    $ ssh private-ip-of-target-instance
    ```
    - Add the following group content to the hosts file - change the ip to the private ip of your target instance 

    ```yml
    [my_group]
    XX.XX.XX.XX ansible_ssh_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa  ansible_python_interpreter=/usr/bin/python3
    ```
    - Now go to your playbooks directory to create a new playbook
    ```zsh
    $ cd ~/playbooks

    $ sudo vi install.xml
    ```
    ```yml
    # Paste the content below or your desired playbook

    ---
    - hosts: Apache_Group
    become: true
    tasks:
        - name: Install apcahe
        apt: name=apache2 state=present update_cache=yes

        - name: ensure apache started
        service: name=apache2 state=started enabled=yes
    ```
    - Run the playbook
    ``` zsh
    $ ansible-playbook installApache.xml
    ```