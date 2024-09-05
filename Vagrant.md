# Provisioning machines locally with Ansible and Vagrant

### Prerequisites and VM set up
First we install Ansible, Vagrant and VirtualBox.
```
sudo apt install ansible vagrant virtualbox -y
```

After the installation is completed, we can get straight to downloading a system image.
```
vagrant box add generic/fedora33
```
![image](https://github.com/user-attachments/assets/828fc2b5-7e20-44b1-b67f-b95392e12c73)

The next step is creating a template by generating a VagrantFile.
We can also write it from scratch in Ruby.
The file contains instructions on setting up the maschines, networks and so on.
```
vagrant init generic/fedora33
```
![image](https://github.com/user-attachments/assets/1a2eed36-f3db-43eb-b39f-8cf524ce8338)

The contents of the file are these, if we ignore all the comments that contain instructions.
Also I added the port forwarding, as it is important for the upcoming steps.
```
Vagrant.configure("2") do |config|
  config.vm.box = "generic/fedora33"
  config.vm.network "forwarded_port", guest: 8080, host: 8080 
end
```

Now we can run the virtual machine and connect to it using these commands.
```
vagrant up
vagrant ssh
```
![image](https://github.com/user-attachments/assets/c451e165-9ddb-4d66-ac24-6a11ff1a82d6)


### Ansible provisioning and Docker set up
After making sure that the virtual machine works, we can get to the proviosioning via Ansible part.
First of all we have to modify the VagrantFile.
```
Vagrant.configure("2") do |config|
  config.vm.box = "generic/fedora33"
  config.vm.network "forwarded_port", guest: 8080, host: 8080 
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
```
![image](https://github.com/user-attachments/assets/5c26e9ba-5ba5-4c2c-87b6-7b999fe49aac)

After this modification we should define the playbook.yml file.
In this example we are installing software to all the available machines.
The tasks will be executed as root because we want to install additional packages.
The only task that is defined is "Install packages".
```
touch playbook.yml
nano playbook.yml
```
```
---
- hosts: all
  become: yes

  tasks:
    - name: "Install packages"
      dnf: "name={{ item }} state=present"
      with_items:
        - nginx
        - certbot
        - postgresql
        - postgresql-server
```

After defining the Ansible file we can now run the virtual machine and provision it from the start.
```
vagrant up --provision
```
![image](https://github.com/user-attachments/assets/d9a7d16b-d5f0-4716-b265-a9504a9a18e9)

The provisioning was a success.
![image](https://github.com/user-attachments/assets/6a0c56f9-c477-4094-a00a-4f2f9ba38c8a)

Now that the instroduction is over we can move on to something more complex.
We will try to install and set up a docker for our virtual machine using Ansible provisioning.
If we want to accomplish this, we need to modify the playbook.yml file.
```
---
- hosts: all
  become: yes

  tasks:
    - name: "Install required packages"
      dnf: 
        name: "{{ item }}" 
        state: present
      with_items:
        - nginx
        - certbot
        - postgresql
        - postgresql-server

    - name: "Remove old versions of Docker if any"
      dnf: 
        name: 
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
        state: absent

    - name: "Install required system packages"
      dnf:
        name: dnf-plugins-core
        state: present

    - name: "Add Docker CE repository"
      command: >
        dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
      args:
        creates: /etc/yum.repos.d/docker-ce.repo

    - name: "Install Docker"
      dnf:
        name: 
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present

    - name: "Start Docker service"
      systemd:
        name: docker
        state: started
        enabled: yes
```

Firstly we remove old versions of Docker, ina case there are any.
Then we install `dnf-plugin-cores` whisch is necessary for managing repositories.
We add the official Focker CE repo to the system and then install Docker CE, Docker CLI, and `containerd.io`.
The last step starts the Docker service and enables it to start on boot.
![image](https://github.com/user-attachments/assets/ee2f9544-aa02-4566-b7dd-8c6e4adfaca2)

Now we can connect to the VM and check if the installation was completed.
```
dnf list installed nginx certbot postgresql postgresql-server docker-ce docker-ce-cli containerd.io
systemctl status docker
```
![image](https://github.com/user-attachments/assets/4ee3679c-f817-42ee-a022-76fe793a2cfa)

### Docker Compose test environment set up

Now that the docker service is up and running, we will create a test environment that contains a web and database service.
The database we will use is PostgreSQL.
For now we will create a docker-compose.yml file that has an Nginx and PostgreSQL service.

```
version: '3'
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./html:/usr/share/nginx/html
    depends_on:
      - db

  db:
    image: postgres:latest
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```
Of course the file should be in the same directory as the playbook.yml and VagrantFile.

![image](https://github.com/user-attachments/assets/fedd7685-f296-4d63-b6ba-46a828365431)

Now it is time to modify the playbook.

```
---
- hosts: all
  become: yes

  tasks:
    - name: "Install required packages"
      dnf:
        name: "{{ item }}"
        state: present
      with_items:
        - nginx
        - certbot
        - postgresql
        - postgresql-server

    - name: "Remove old versions of Docker if any"
      dnf:
        name:
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
        state: absent

    - name: "Install required system packages"
      dnf:
        name: dnf-plugins-core
        state: present

    - name: "Add Docker CE repository"
      command: >
        dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
      args:
        creates: /etc/yum.repos.d/docker-ce.repo

    - name: "Install Docker"
      dnf:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present

    - name: "Start Docker service"
      systemd:
        name: docker
        state: started
        enabled: yes

    - name: "Install Docker Compose"
      get_url:
        url: https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Linux-x86_64
        dest: /usr/local/bin/docker-compose
        mode: '0755'

    - name: "Create Docker-Compose directory"
      file:
        path: /opt/docker-compose
        state: directory
        mode: '0755'

    - name: "Ensure nginx.conf is a file"
      file:
        path: /opt/docker-compose/nginx.conf
        state: touch
        mode: '0644'

    - name: "Ensure nginx.conf is configured correctly"
      copy:
        dest: /opt/docker-compose/nginx.conf
        content: |
          user  nginx;
          worker_processes  auto;
          error_log  /var/log/nginx/error.log warn;
          pid        /var/run/nginx.pid;

          events {
              worker_connections  1024;
          }

          http {
              include       /etc/nginx/mime.types;
              default_type  application/octet-stream;

              log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                                '$status $body_bytes_sent "$http_referer" '
                                '"$http_user_agent" "$http_x_forwarded_for"';

              access_log  /var/log/nginx/access.log  main;

              sendfile        on;
              keepalive_timeout  65;

              include /etc/nginx/conf.d/*.conf;
          }

    - name: "Create Docker-Compose file for full-fledged environment"
      copy:
        dest: /opt/docker-compose/docker-compose.yml
        content: |
          version: '3'
          services:
            web:
              image: nginx:latest
              ports:
                - "8080:80"
              volumes:
                - ./nginx.conf:/etc/nginx/nginx.conf
                - ./html:/usr/share/nginx/html
              depends_on:
                - db

            db:
              image: postgres:latest
              environment:
                POSTGRES_DB: testdb
                POSTGRES_USER: testuser
                POSTGRES_PASSWORD: testpass
              volumes:
                - pgdata:/var/lib/postgresql/data

          volumes:
            pgdata:

    - name: "Ensure html directory exists"
      file:
        path: /opt/docker-compose/html
        state: directory
        mode: '0755'

    - name: "Ensure html directory has correct permissions"
      file:
        path: /opt/docker-compose/html
        state: directory
        recurse: yes
        mode: '0755'

    - name: "Create index.html file"
      copy:
        dest: /opt/docker-compose/html/index.html
        content: |
          <html>
            <head><title>Welcome</title></head>
            <body><h1>Welcome to Nginx</h1></body>
          </html>
        mode: '0644'

    - name: "Start Docker-Compose service"
      command: docker-compose up -d
      args:
        chdir: /opt/docker-compose
```

I have encountered an error with tne nginx.conf file, this is why I have created a task that verifies the creation of that file, in case it doesn't exist.
Then I modify the file so it is configured correctly.
Also there was a problem eith the permissions for the default /html location. I also took care of them.
The detailed description of both the YAML files is down bellow.

Now that everything is ready we can try provisioning our VM.

![image](https://github.com/user-attachments/assets/d3e2a0b1-e311-40b9-a2ef-2b5f3a2d5949)

The provisioning was a success.
Now we will connect to the VM and test if everything works.

![image](https://github.com/user-attachments/assets/42666546-a1a4-42b6-b278-94a406843f13)

As we can see the test environment was created and the services are up and running.
Now we will try to connect to the database inside the test environment.

![image](https://github.com/user-attachments/assets/62642b02-2cd2-4dc9-acfb-af235769c09a)

We have created the testdb and the testuser, having superuser priveleges.

Now let's see if the Nginx service is working properly.

![image](https://github.com/user-attachments/assets/38241ea1-96b1-433c-9eff-519f6ef76b79)

The curl command works on the VM. Let's try accessing the server from our host machine.

![image](https://github.com/user-attachments/assets/a47d9686-2a7a-4910-9986-15f7fb94903c)

And it works!

## YAML files explanation

### `docker-compose.yml`

```
version: '3'  # Specifies the Docker Compose version to use

services:
  web:  # Defines the 'web' service
    image: nginx:latest  # Uses the latest Nginx image from Docker Hub
    ports:
      - "8080:80"  # Maps port 80 of the container to port 8080 on the host
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf  # Mounts the local 'nginx.conf' file to the container's Nginx configuration path
      - ./html:/usr/share/nginx/html  # Mounts the local 'html' directory to the container's web root
    depends_on:
      - db  # Specifies that the 'web' service depends on the 'db' service

  db:  # Defines the 'db' service
    image: postgres:latest  # Uses the latest Postgres image from Docker Hub
    environment:
      POSTGRES_DB: testdb  # Sets the name of the default database
      POSTGRES_USER: testuser  # Sets the username for the default database
      POSTGRES_PASSWORD: testpass  # Sets the password for the default database user
    volumes:
      - pgdata:/var/lib/postgresql/data  # Mounts a named volume for persistent database storage

volumes:
  pgdata:  # Defines the 'pgdata' named volume for the Postgres database
```

### playbook.yml
```
---
- hosts: all  # Target all hosts defined in the inventory
  become: yes  # Use sudo to run tasks with elevated privileges

  tasks:
    - name: "Install required packages"
      dnf:
        name: "{{ item }}"  # Specify the package name to be installed
        state: present  # Ensure the package is installed
      with_items:
        - nginx  # Install Nginx web server
        - certbot  # Install Certbot for obtaining SSL certificates
        - postgresql  # Install PostgreSQL client
        - postgresql-server  # Install PostgreSQL server

    - name: "Remove old versions of Docker if any"
      dnf:
        name:
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
        state: absent  # Ensure old or conflicting Docker packages are removed

    - name: "Install required system packages"
      dnf:
        name: dnf-plugins-core  # Install DNF plugins for additional package management functionality
        state: present

    - name: "Add Docker CE repository"
      command: >
        dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
      args:
        creates: /etc/yum.repos.d/docker-ce.repo  # Run this command only if the Docker repository file does not already exist

    - name: "Install Docker"
      dnf:
        name:
          - docker-ce  # Install Docker Community Edition
          - docker-ce-cli  # Install Docker CLI
          - containerd.io  # Install containerd, which Docker depends on
        state: present  # Ensure Docker packages are installed

    - name: "Start Docker service"
      systemd:
        name: docker  # Specify the Docker service
        state: started  # Start the Docker service
        enabled: yes  # Enable Docker to start on system boot

    - name: "Install Docker Compose"
      get_url:
        url: https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Linux-x86_64
        dest: /usr/local/bin/docker-compose  # Download Docker Compose binary to the specified location
        mode: '0755'  # Set executable permissions for the Docker Compose binary

    - name: "Create Docker-Compose directory"
      file:
        path: /opt/docker-compose  # Create a directory for Docker Compose files
        state: directory
        mode: '0755'  # Set directory permissions

    - name: "Ensure nginx.conf is a file"
      file:
        path: /opt/docker-compose/nginx.conf  # Ensure the Nginx configuration file exists
        state: touch
        mode: '0644'  # Set permissions for the Nginx configuration file

    - name: "Ensure nginx.conf is configured correctly"
      copy:
        dest: /opt/docker-compose/nginx.conf  # Copy the Nginx configuration to the specified path
        content: |
          user  nginx;
          worker_processes  auto;
          error_log  /var/log/nginx/error.log warn;
          pid        /var/run/nginx.pid;

          events {
              worker_connections  1024;
          }

          http {
              include       /etc/nginx/mime.types;
              default_type  application/octet-stream;

              log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                                '$status $body_bytes_sent "$http_referer" '
                                '"$http_user_agent" "$http_x_forwarded_for"';

              access_log  /var/log/nginx/access.log  main;

              sendfile        on;
              keepalive_timeout  65;

              include /etc/nginx/conf.d/*.conf;
          }

    - name: "Create Docker-Compose file for full-fledged environment"
      copy:
        dest: /opt/docker-compose/docker-compose.yml  # Create a Docker Compose configuration file
        content: |
          version: '3'
          services:
            web:
              image: nginx:latest  # Use the latest Nginx image from Docker Hub
              ports:
                - "8080:80"  # Map port 80 in the container to port 8080 on the host
              volumes:
                - ./nginx.conf:/etc/nginx/nginx.conf  # Mount local Nginx configuration file into the container
                - ./html:/usr/share/nginx/html  # Mount local HTML directory into the container
              depends_on:
                - db  # Ensure the 'db' service is started before 'web'

            db:
              image: postgres:latest  # Use the latest PostgreSQL image from Docker Hub
              environment:
                POSTGRES_DB: testdb  # Set the name of the default database
                POSTGRES_USER: testuser  # Set the username for the database
                POSTGRES_PASSWORD: testpass  # Set the password for the database user
              volumes:
                - pgdata:/var/lib/postgresql/data  # Mount named volume for PostgreSQL data

          volumes:
            pgdata:  # Define the named volume for PostgreSQL data

    - name: "Ensure html directory exists"
      file:
        path: /opt/docker-compose/html  # Ensure the 'html' directory exists
        state: directory
        mode: '0755'

    - name: "Ensure html directory has correct permissions"
      file:
        path: /opt/docker-compose/html  # Set permissions for the 'html' directory and its contents
        state: directory
        recurse: yes
        mode: '0755'

    - name: "Create index.html file"
      copy:
        dest: /opt/docker-compose/html/index.html  # Create an 'index.html' file with a welcome message
        content: |
          <html>
            <head><title>Welcome</title></head>
            <body><h1>Welcome to Nginx</h1></body>
          </html>
        mode: '0644'

    - name: "Start Docker-Compose service"
      command: docker-compose up -d  # Start the Docker Compose services in detached mode
      args:
        chdir: /opt/docker-compose  # Change to the Docker Compose directory before running the command
```

## Useful commands
- `vagrant up` that will instruct Vagrant to create and run the virtual machine
- `vagrant halt` that will stop the virtual machine
- `vagrant destroy` that will destroy the virtual machine (useful when we change the VagrantFile and want to recreate it again)
- `vagrant ssh` that will connect to that machine and give us a remote command line session
- `vagrant ssh-config` that will display SSH connection information if we want to connect to the machine without Vagrant

## Useful links
- https://developer.hashicorp.com/vagrant/tutorials
- https://stribny.name/blog/provisioning-ansible-vagrant/
- https://github.com/ttskch/ansible-docker/blob/master/playbook.yml
