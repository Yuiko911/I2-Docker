# Discover Ansible
### Question 3.1
**Document your inventory and base commands**

```yml
all:
 vars:
   ansible_user: admin
   ansible_ssh_private_key_file: /home/yuuka/Documents/EFREI/I2-Docker/TP3/ansible/inventories/id_rsa 
 children:
   prod:
     hosts: antoine.meunier.takima.cloud 
```

