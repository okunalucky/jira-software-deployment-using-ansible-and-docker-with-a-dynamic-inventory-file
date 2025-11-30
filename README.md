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
