# Machine-Test- Output with all steps and commands

###### 1st Step
I have created a new 4 instance in AWS account for configurning the docker and swarm, These instances are Ubuntu and the Version is 20.04. I have used 1 instance for Master and other 3 for worker nodes.

![Screenshot from 2024-02-17 21-50-35](https://github.com/abhirajparthan/Machine-Test/assets/100773790/7ac8f8a9-7134-4ff6-b436-183116afb0b2)

----

###### 2nd Step 

# 1. Install docker and set up docker swarm.

I have Installed the docker in 4 Instance using the below command. 
~~~
apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt install docker-ce
systemctl status docker
~~~

![Screenshot from 2024-02-16 17-33-38](https://github.com/abhirajparthan/Machine-Test/assets/100773790/6a30e100-956d-4abf-b675-9cc65363743a)


Also enabled the docker service in the 4 Instance using the command ( Its for once the instance is stop/start done. The docker service will start automatically). 
~~~
systemctl enable docker
~~~

![Screenshot from 2024-02-16 17-38-44](https://github.com/abhirajparthan/Machine-Test/assets/100773790/90736822-534b-4fea-83bb-5e42dba9cd87)

I have created the docker swarm cluster using the command. 

~~~
docker swarm init
~~~

![Screenshot from 2024-02-16 17-44-51](https://github.com/abhirajparthan/Machine-Test/assets/100773790/2d8962d4-b10b-4293-97d4-70cb1e2086dc)

After that we will get the tocken as output in the screen and we need to run this tocken in every nodes.

Then We can see the worker nodes and master nodes using the command.

~~~
docker node ls
~~~

![Screenshot from 2024-02-18 15-11-42](https://github.com/abhirajparthan/Machine-Test/assets/100773790/a9340d24-60f2-4d76-bc63-f68459730e2e)

-----
You can create the worker tocken after the 10 days using the below command.
~~~
docker swarm join-token worker
~~~
-----

##### 3ed Step

# 2. Deploy a simple docker stack which contains two services (use compose file). 

----

  ##### 1. MySQL service with volume.

Setup a volume. I have created EFS service in AWS ( Its similaer to NFS ) and it mounted to worker nodes. 

![Screenshot from 2024-02-18 15-12-22](https://github.com/abhirajparthan/Machine-Test/assets/100773790/21801367-fb91-4780-b92a-09562b1c510f)

Then Created a 1 folders in the efs. The folder names is mysql. 

![Screenshot from 2024-02-18 15-14-03](https://github.com/abhirajparthan/Machine-Test/assets/100773790/ed937358-f28a-4567-a607-1621182d2ed3)

-------

For deploying the stack . I have created a folder flask_app in the manager node and created a docker-compose.yml file inside the folder. Here I am using mysql:5.7 image for database. Also I mounted the container /var/lib/mysql to efs folder( Here I am using bind mount. ).  Then added the resource label of 2 nodes is flask and 1 is mysql
~~~
vi docker-compose.yml

---

version: '3.8'

services:

  database:
    image: mysql:5.6
    networks:
      - mysql_net
    volumes:
      - type: bind
        source: /root/volume/mount/mysql
        target: /var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: qwerty123!
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    ports:
      - "3306:3306"
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.resource == mysql
    healthcheck:
      test: "mysqladmin ping -h 127.0.0.1 -u root --password=qwerty123! || exit 1"
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

networks:
  mysql_net:
    driver: overlay
~~~

I have deployed the stack using the command, My stack name in abhi
~~~
docker stack deploy -c docker-compose.yml abhi
~~~

After the deploying I have check the mysql replica using command 
~~~
docker service ls
~~~

![Screenshot from 2024-02-18 14-47-12](https://github.com/abhirajparthan/Machine-Test/assets/100773790/f2e50154-fa7b-43e5-936e-7e22a669c510)

Also I checked the database and user name created insdie the container depends on the env, Its created 

![Screenshot from 2024-02-17 12-14-28](https://github.com/abhirajparthan/Machine-Test/assets/100773790/3dc8540e-bd54-4d01-8149-9c451ce4181f)

Then checked the volume is mounted properly. Checked the mounted path /root/volume/mount/mysql I can see that the mounted files in the location.

![Screenshot from 2024-02-18 15-14-45](https://github.com/abhirajparthan/Machine-Test/assets/100773790/ad900da4-afdb-4ee2-8a45-6838aa7f133c)

----

  ##### 2. Simple app service (you can find some simple Flask apps from communities) which should connect to the above MySQL.

  ----
  
For building the Flask application I have created 3 files in the local directory. The files are app.py, requirement.txt, Dockerfile.

![Screenshot from 2024-02-17 14-13-59](https://github.com/abhirajparthan/Machine-Test/assets/100773790/699bcee1-7272-490a-ac60-550faabfce2c)

I have importe the code to app.py file: It connects to a MySQL database and retrieves data from the employee_data table. The data is returned as a JSON response when accessing the root URL.

~~~
vi app.py

from flask import Flask, jsonify
import mysql.connector


app = Flask(__name__)


def employee_data():
    config = {
        'user': 'wordpress',
        'password': 'wordpress',
        'host': '172.31.23.66', #added Private IP of the server
        'port': '3306',
        'database': 'wordpress'
    }
    connection = mysql.connector.connect(**config)
    cursor = connection.cursor(dictionary=True)
    cursor.execute('SELECT Employee_Name, Title FROM employee_data')
    results = cursor.fetchall()
    cursor.close()
    connection.close()
    return results


@app.route('/')
def index():
    return jsonify({'Employee Data': employee_data()})


if __name__ == '__main__':
    app.run(host='0.0.0.0')
~                                                                                                                                             
~                                 
~~~
 
Then Import the requirement packages to requirement.txt
~~~
vi requirement.txt 

Flask
mysql-connector
~~~

Then I have writed the Dockerfile for building the docker Image.

~~~
vi Dockerimage

FROM python:3.6

WORKDIR /app

COPY . /app

RUN pip install -r requirements.txt

EXPOSE 5000

CMD python app.py
~~~

Then I build the Image using the command 

~~~
sudo docker image build -t flaskbhi:latest .
~~~

![Screenshot from 2024-02-17 14-21-35](https://github.com/abhirajparthan/Machine-Test/assets/100773790/e3faaf8d-b8ca-4b04-bf5b-26fafe5e2059)

Then I uploaded the docker image to docker hub. ( Changed the tag, login to the hub and then pushed the latest image )

![Screenshot from 2024-02-17 14-27-21](https://github.com/abhirajparthan/Machine-Test/assets/100773790/4a8c04d3-4d33-4803-b3eb-9fddc904ef61)

------------

For the database connectivity, I have imported table and table details to the mysql database. 

![Screenshot from 2024-02-18 14-22-23](https://github.com/abhirajparthan/Machine-Test/assets/100773790/309d0b2b-b1f3-4536-a111-910a321f90b2)

Then modified the exsisting docker-compose.yml for the flask application

~~~
vi docker-compose.yml

---

version: '3.8'

services:

  database:    
    image: mysql:5.7 
    networks:
       - mysql_net
    volumes:
      - type: bind
        source: /root/volume/mount/mysql
        target: /var/lib/mysql    
    environment:
      MYSQL_ROOT_PASSWORD: qwerty123!
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
    ports:
      - "3306:3306"
    deploy:
      replicas: 1
      placement:
        constraints: 
          - node.labels.resource == mysql
    healthcheck:
      test: "mysqladmin ping -h 127.0.0.1 -u root --password=qwerty123! || exit 1"
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

  flask:
    image: aparthan/flask_machine:latest
    networks:
      - flask_net
      - mysql_net
    ports:
      - "5000:5000"
    depends_on:
      - database
    deploy:
      replicas: 7
      placement:
        constraints:
          - node.labels.resource == flask

networks:
  flask_net:
    driver: overlay
  mysql_net:
    driver: overlay

~~~

After the I have deployed the flask application with mysql usig the command 

![Screenshot from 2024-02-17 21-57-49](https://github.com/abhirajparthan/Machine-Test/assets/100773790/7984e621-2170-416e-9b45-c00cadf5f989)

Then I have checked the services using the command 

![Screenshot from 2024-02-17 22-01-30](https://github.com/abhirajparthan/Machine-Test/assets/100773790/056e6726-e555-4625-ab4d-784b42b2e4b1)


Now the stack deployment is completed. We can acces the flask applicaton from the blow ip with port

~~~
http://18.222.172.21:5000/
http://18.118.6.242:5000/
~~~

We can see that the out put is below.

![Screenshot from 2024-02-18 14-25-36](https://github.com/abhirajparthan/Machine-Test/assets/100773790/580c491e-4bfd-4a67-b4fd-abbbd7bd2fac)

![Screenshot from 2024-02-18 14-25-42](https://github.com/abhirajparthan/Machine-Test/assets/100773790/44fbe938-1d9d-4585-b028-d917fc3b4fa9)


This Outpot is Similar to the databse table. We can confirm the database is connected to the flask application

-----

# Deploy another docker stack with traefik service. The request for the above app should be load balance through traefik service.

I have created a folder for traefik and created a docker-compose.yml file for stack creation

~~~
version: '3.8'

services:
  traefik:
    image: traefik:v2.5
    networks:
       - traefik_net
    command:
      - "--api.insecure=true"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - type: bind
        source: /var/run/docker.sock
        target: /var/run/docker.sock

    deploy:
      placement:
        constraints:
          - node.role == manager

  web:
    image: nginx:alpine
    networks:
       - traefik_net
    volumes:
      - type: bind
        source: /root/volume/mount/web/default.conf
        target: /etc/nginx/conf.d/default.conf 
    deploy:
      replicas: 3
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.myapp.rule=Host(`abhiraj.ga`)"
        - "traefik.http.services.myapp.loadbalancer.server.port=80"

networks:
  traefik_net:

~~~
I have mounted the nginx default configuration file to the efs volume and the default configuration file content is below. We can load balance the stack node here. Here we I am using the private IP for the node server for upstream

~~~
upstream samplecluster {
  server 172.31.18.254:5000 weight=50;
  server 172.31.26.85:5000 weight=50;
}

server {
  listen 80;
  server_name abhiraj.ga;

  location / {
    proxy_pass http://samplecluster;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
~~~
I have added my domain name as abhiraj.ga and its entry addded to my laptop /etc/hosts file. ( For accesing the domain in browser. )

![Screenshot from 2024-02-18 12-54-01](https://github.com/abhirajparthan/Machine-Test/assets/100773790/86e93248-f06e-46b1-9696-d66bb76164c9)

I have deployed the terafik stack using the below command.

![Screenshot from 2024-02-18 13-04-12](https://github.com/abhirajparthan/Machine-Test/assets/100773790/cb627cda-c5e2-4f85-9094-719e381098be)

![Screenshot from 2024-02-18 13-04-47](https://github.com/abhirajparthan/Machine-Test/assets/100773790/5bdd5014-3d48-43f7-8fb5-52bab580f9fa)

I have try to access the domain abhiraj.ga with ports 8080 and 80 in the browser and I got the traefik dash board in abhiraj.ga:8080.

![Screenshot from 2024-02-18 13-07-14](https://github.com/abhirajparthan/Machine-Test/assets/100773790/89dbfb06-ce27-40de-b08b-3dea54cb8361)

I got the application with url abhiraj.ga domain in the browser.

![Screenshot from 2024-02-18 12-49-56](https://github.com/abhirajparthan/Machine-Test/assets/100773790/3c41f116-1737-4fa6-b5a0-27e06043ab2e)


-----------------

#### Thank you for your time and consideration :) 
