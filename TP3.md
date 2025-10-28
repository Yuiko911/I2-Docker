# Discover Ansible
### Question 3.1
**Document your inventory and base commands**

```yml
all:
  vars:
	# Name of our user
    ansible_user: admin

	# Link to our private key
    ansible_ssh_private_key_file: /home/yuuka/Documents/EFREI/I2-Docker/TP3/ansible/inventories/id_rsa 
  children:
    prod:
	  # List of all our servers
      hosts: antoine.meunier.takima.cloud 
```

```sh
# Check that all of the hosts are running
ansible all -i inventories/setup.yml -m ping
# Install apache2 on all hosts
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```

### Question 3.2
**Document your playbook**

```yml
# Execute on all hosts
- hosts: all   
  gather_facts: true
  # Execute as root
  become: true

  # Execute each role, in order
  roles:
    # Install Docker
    - docker
    # Setup volumes
    - volume
    # Setup networks
    - network
    # Start the database container
    - database
    # Start the API container
    - app
    # Start the proxy container
    - proxy
```

### Question 3.3
**Document your docker_container tasks configuration**

#### Docker
This file is already selfl-documented through comments.

#### Volume
```yml
- name: Create Docker volume for PostgreSQL
  community.docker.docker_volume:
    name: postgres-data
    state: present
```
We use the community.docker.docker_volume module to create a volume called "postgres-data", and start is with `state: present`

#### Network
```yml
- name: Create Docker app-network
  community.docker.docker_network:
    name: app-network
    state: present

- name: Create Docker proxy-network
  community.docker.docker_network:
    name: proxy-network
    state: present
```
Here, we use the `community.docker.docker_network` module to create networks called `app-network` and `proxy-network`, and start is with `state: present`

#### Database
```yml
- name: Launch database
  community.docker.docker_container:
    name: database
	# Name of the image
    image: ilovedockingandfrotting/tp-devops-database:latest
	# Always restart the container if it were to fail, unless we did so manually
    restart_policy: unless-stopped
	# Always re-pull the image when the playbook is launched
    pull: always
	
	# Volume used
    volumes:
      - postgres-data:/var/lib/postgresql/data
	# Network used
    networks:
      - name: app-network
	# Environemment variables
    env:
      POSTGRES_DB: "{{ DB_NAME }}"
      POSTGRES_USER: "{{ DB_USERNAME }}"
      POSTGRES_PASSWORD: "{{ DB_PASSWORD }}"
```

#### API
```yml
- name: Launch API
  community.docker.docker_container:
    name: app
	# Name of the image
    image: ilovedockingandfrotting/tp-devops-simple-api:latest
	# Always re-pull the image when the playbook is launched
    pull: always
	# We restart the container a maximum of 3 times if it fails
    restart_policy: on-failure
	restart_retries: 3
	# Network used
    networks:
      - name: app-network
      - name: proxy-network
	# Environemment variables
    env:
      DB_SERVER: "database"
      DB_NAME: "{{ DB_NAME }}"
      DB_USERNAME: "{{ DB_USERNAME }}"
      DB_PASSWORD: "{{ DB_PASSWORD }}"
```

#### Proxy
```yml
- name: Run HTTPD
  docker_container:
    name: httpd
	# Name of the image
    image: ilovedockingandfrotting/tp-devops-httpd2:latest
	# Always re-pull the image when the playbook is launched
    pull: always
	# We never restart the container
    restart_policy: no
	# Network used
    networks:
      - name: proxy-network
	# We give a public port for this container, as it is the one that clients will connect to.
    ports:
      - "80:80"
	# Environemment variables
    env:
      PAGE_NAME: "app"
```
