# jira-software-deployment-using-ansible-and-docker-with-a-dynamic-inventory-file
- Create an instance called Control which serves as the master node;
specifications:amazon linux 2023 ami, t2.micro, set the inboud traffic rule and under advanced data, select amazon ec2 full access
- Create an instance called Webservers which serves as the slave node;
under tags, click on manage tags and name it according to the screenshot below:
<img width="1208" height="122" alt="Image" src="https://github.com/user-attachments/assets/1be127d1-863d-4d96-9709-25d9b1e1f084" />
specifications:amazon linux 2023 ami, t2.micro, set the inboud traffic rule and launch the instance
- Log into the Control, change to root user, add ansible as a user and set a password for it

```sh

sudo su
useradd ansible
passwd ansible

```

- Allow for password aunthentication and permit root login for ansible user 
```sh
sudo nano /etc/ssh/sshd_config  
```
- Uncomment permit rootlogin and change password aunthentication to yes
- Restart the sshd
```sh
sudo systemctl restart sshd
```
- Add ansible to the sudoers group
 ```sh
sudo nano /etc/sudoers
```
- Navigate to the home directory of the ansible user, update dnf. install ansiblr, boto 3, python 3 and ansible 
```sh
 cd ~
sudo dnf update -y
sudo dnf install -y ansible-core python3-pip jq awscli
python3 -m pip install --user boto3 botocore
ansible-galaxy collection install amazon.aws
```    
- Create the .ssh directory
```sh
mkdir ~/.ssh
```
- Long list to check the file permission

```sh
ls -la
```
- Change the file permission using:
 
 ```sh
chmod 700 ~/.ssh
```
```sh
cd .ssh/
```
- Copy and paste the key pair in the worker node into myteam.pm
```sh
sudo nano myteam.pem
```
- Change the file permissions
```sh
sudo chmod 600 myteam.pem
sudo chown ansible:ansible /home/ansible/.ssh/myteam.pem
sudo chmod 400 / home/ansible/.ssh/myteam.pem
```
- Create a directory called ansible_aws and follow the commands below:
```sh
mkdir ansible_aws
cd ansible_aws
mkdir inventory
ls
cd inventory/
sudo nano aws_ec2.yml
```
- Paste the code below into the aws_ec2.yml
```yaml
plugin: amazon.aws.aws_ec2

regions:
  - us-east-1

filters:
  instance-state-name: running
  "tag:Name": Webservers
  "tag:Env": Production

hostnames:
  - tag:Name

compose:
  ansible_host: private_ip_address

keyed_groups:
  - key: tags.Env
    prefix: tag_Env
  - key: tags.Name
    prefix: tag_Name

strict: False

```
- Create the ansible.cfg file
```sh
cd ~
cd ansible_aws/
sudo nano ansible.cfg
```
- Paste in the code for the ansible.cfg

```yaml
[defaults]
inventory= ./inventory/aws_ec2.yml
host_key_checking = False
collections_paths = ~/.ansible/collections:/usr/share/ansible/collections
ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
```
- Test if the inventory file will automatically pick the instance in the aws environment
```sh
cd ~ 
cd ansible_aws/
ansible-inventory -i ~/ansible_aws/inventory/aws_ec2.yml --list
```
- Create the docker compose file
```sh
sudo nano docker-compose.yml
```
- Paste the code into docker-compose.yml

```sh
version: '3'
services:
  jira:
    image: atlassian/jira-software:9.15  # Use a stable version, not "latest"
    depends_on:
      - db
    ports:
      - "8080:8080"
    environment:
      ATL_JDBC_URL: jdbc:postgresql://db:5432/jiradb
      ATL_JDBC_USER: jirauser
      ATL_JDBC_PASSWORD: jirapassword
      ATL_DB_TYPE: postgres72
    volumes:
      - jira-data:/var/atlassian/jira

  db:
    image: postgres:12      # Jira 9 supports PostgreSQL 10–14 (NOT 9.6)
    environment:
      POSTGRES_DB: jiradb
      POSTGRES_USER: jirauser
      POSTGRES_PASSWORD: jirapassword
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U jirauser"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  jira-data:
  db-data:
```
- Create the playbook deploy_jira.yml
 ```sh
sudo nano deploy_jira.yml
```
- Paste in the code below
```yaml
---
- name: Deploy Jira via Docker Compose on Amazon Linux 2023
  hosts: "tag_Name_Webservers:&tag_Env_Production"
  gather_facts: yes
  become: true
  vars:
    ansible_user: ec2-user
    ansible_python_interpreter: /usr/bin/python3
    jira_project_dir: /opt/jira
    compose_version: "v2.27.1"
    compose_url: "https://github.com/docker/compose/releases/download/{{ compose_version }}/docker-compose-linux-x86_64"

  tasks:
    # 1. Install Docker engine
    - name: Install Docker engine
      ansible.builtin.dnf:
        name: docker
        state: present

    # 2. Ensure Docker service is started and enabled
    - name: Ensure Docker service is running
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    # 3. Create CLI plugin directory for Docker Compose v2
    - name: Create CLI plugin directory for Docker Compose v2
      ansible.builtin.file:
        path: /usr/local/lib/docker/cli-plugins
        state: directory
        mode: '0755'

    # 4. Install Docker Compose v2 plugin
    - name: Install Docker Compose v2 plugin
      ansible.builtin.get_url:
        url: "{{ compose_url }}"
        dest: /usr/local/lib/docker/cli-plugins/docker-compose
        mode: '0755'

    # 5. Create Jira project directory
    - name: Create Jira project directory
      ansible.builtin.file:
        path: "{{ jira_project_dir }}"
        state: directory
        mode: '0755'

    # 6. Copy your Compose file from control node to target node
    - name: Copy Docker Compose file from local to remote
      ansible.builtin.copy:
        src: "{{ lookup('env', 'HOME') }}/ansible_aws/docker-compose.yml"
        dest: "{{ jira_project_dir }}/docker-compose.yml"
        mode: '0644'

    # 7. Deploy Jira stack using Docker Compose v2
    - name: Deploy Jira stack
      community.docker.docker_compose_v2:
        project_src: "{{ jira_project_dir }}"
        state: present
```
- Configure the Webservers
```sh
sudo nano web.yml
```
- Paste in the code below:
  
```yaml
---
- name: Configure Production Webservers
  hosts: "tag_Name_Webservers:&tag_Env_Production"
  gather_facts: yes
  become: yes
  vars:
    ansible_user: ec2-user
    ansible_python_interpreter: /usr/bin/python3
  tasks:
    - name: Ensure Nginx is installed
      ansible.builtin.package:
          name: nginx
          state: present
    - name: Start Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```
- Install the latest ansible galaxy collection
```sh
ansible-galaxy collection install community.docker --force
```
- Run the playbook
  
```sh
ansible-playbook -i ansible_aws/inventory/aws_ec2.yml ansible_aws/deploy_jira.yml -u ec2-user --private /home/ansible/.ssh/myteam.pem -vvv
```
<img width="1240" height="658" alt="Image" src="https://github.com/user-attachments/assets/5db562fb-85ae-47be-999e-83338511c6bb" />
# jira set up
