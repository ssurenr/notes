# Technical Report: Migrating a Django Application to an On-Premise Environment

## Server Setup and Initial Configuration

### Platform Details

|               |                 |
|---------------|-----------------|
| IaaS Provider | Digital Ocean   |
| IaaS Type     | Virtual machine |
| OS            | RHEL 9 (Rocky)  |


### System Preparation

1. **SSH Configuration**:
   - Create SSH directory: `mkdir ~/.ssh`
   - Add authorized keys: `vim ~/.ssh/authorized_keys` and then `chmod 600 ~/.ssh/authorized_keys`

2. **Package Installation and System Update**:
   - Install Vim: `sudo dnf install vim -y`
   - Install Git: `sudo dnf install git -y`
   - Enable CRB and install EPEL: 
     ```bash
     sudo dnf config-manager --set-enabled crb
     sudo dnf install epel-release -y
     ```

3. **Docker Installation**:
   - Add Docker repository: `sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
   - Install Docker: `sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y`
   - Enable and start Docker: `sudo systemctl enable --now docker`
   - Add user to Docker group: `sudo usermod -aG docker $USER`

4. **Docker Configuration**:
   - Configure Docker daemon for log rotation:
     ```bash
     sudo bash -c 'cat << EOF > /etc/docker/daemon.json
     {
       "log-driver": "json-file",
       "log-opts": {
         "max-size": "10m",
         "max-file": "3"
       }
     }
     EOF'
     sudo systemctl restart docker
     ```

### Application Deployment

1. **Repository Cloning**:
   - Install github cli: `dnf install https://github.com/cli/cli/releases/download/v2.45.0/gh_2.45.0_linux_amd64.rpm`
   - Authenticate using Github client: `gh auth login`
   - Clone the application repository: `gh repo clone panmo-us/mars-analytics`

2. **Docker Compose Setup**:
   - Navigate to the application directory: `cd mars-analytics`
   - Edit Docker Compose file as needed: `vim docker-compose.yml`
   ```dockerfile
    version: "3.9"
    services:
      web:
        build:
          context: .
          dockerfile: Dockerfile
        environment:
          - "DJANGO_DEBUG=True"
          - "MODE=dev"
          - "DATABASE_URL=postgres://postgres@db/postgres"
        entrypoint: ""
        command: python manage.py runserver 0.0.0.0:5000
        ports:
          - "5000:5000"
        depends_on:
          - db
      db:
        image: postgres:14-alpine
        volumes:
          - postgres_data:/var/lib/postgresql/data/
        environment:
          - "POSTGRES_HOST_AUTH_METHOD=trust"
    
    volumes:
      postgres_data:
   ```
   - Start the application: `docker compose up -d`
   ```shell
        docker ps
        CONTAINER ID   IMAGE                COMMAND                  CREATED       STATUS         PORTS                                       NAMES
        54610c52da50   mars-analytics-web   "python manage.py ru…"   4 hours ago   Up 4 seconds   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp   mars-analytics-web-1
        c933dc8113fc   postgres:14-alpine   "docker-entrypoint.s…"   6 hours ago   Up 5 seconds   5432/tcp                                    mars-analytics-db-1
   ```

3. **Database Configuration**:
   - Make migrations: `docker compose exec web python manage.py makemigrations confab`
   - Apply migrations: `docker compose exec web python manage.py migrate --run-syncdb`
   - Import initial data: `docker compose exec web python manage.py loaddata ./data/*.yaml`
   - Create Cache Table: `docker compose exec web python manage.py createcachetable`

4. **Superuser Creation**:
   - Create a Django superuser: `docker compose exec web python manage.py createsuperuser`


## Production Configuration (Future Steps)
### Scaling the Architecture

- **Horizontal Scaling**:
  - To add more web server instances, update the `docker-compose.yml` file with additional instances under the web service.
  - Implement a load balancer such as Nginx or HAProxy in front of the web servers to distribute traffic.
  
- **Vertical Scaling**:
  - Upgrade the server hardware or VM resources (CPU, RAM, disk space) as needed to support increased load.

### Securing with SSL

1. **Obtain an SSL Certificate**:
   - Use Let's Encrypt: `sudo certbot certonly --nginx -d example.com`
   
2. **Configure Nginx as a Reverse Proxy with SSL**:
   - Edit Nginx configuration: `sudo vim /etc/nginx/sites-available/example.com`
   - Set up redirection from HTTP to HTTPS and configure the SSL certificate and key paths.

### Serving Static Components with Nginx

- **Configure Nginx to Serve Static Files**:
  - In the Nginx server block, add:
    ```nginx
    location /static/ {
        alias /app/staticfiles/;
    }
    ```
  - Collect static files in Django: `docker compose exec web python manage.py collectstatic`

###  Continuous Delivery
- **Automate Deployment with Github Actions**:
  - We can create a Github Actions workflow to build and deploy the application on push to the main branch.
