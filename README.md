# Machine-Test- Output with all steps and commands

###### 1st Step
I have created a new 4 instance in AWS account for configurung the docker and swarm, These instances are Ubuntu and the Version is 20.04. I have used 1 instance for Master and other 3 for worker nodes.

![Screenshot from 2024-02-17 21-50-35](https://github.com/abhirajparthan/Machine-Test/assets/100773790/7ac8f8a9-7134-4ff6-b436-183116afb0b2)

----

###### 2nd Step 

# 1. Install docker and set up docker swarm.

I have Installed the docker in 3 Instance using the below command. 
~~~
apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt install docker-ce
systemctl status docker
~~~

![Screenshot from 2024-02-16 17-33-38](https://github.com/abhirajparthan/Machine-Test/assets/100773790/6a30e100-956d-4abf-b675-9cc65363743a)


Also I have enabled the docker service in the 3 Instance using the command ( Its for once the instance is stop/start done. The docker service will start automatically). 
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

![Screenshot from 2024-02-17 21-51-52](https://github.com/abhirajparthan/Machine-Test/assets/100773790/d2097ec0-ce89-4623-b422-09c6b9c32f16)

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

![Screenshot from 2024-02-17 10-17-21](https://github.com/abhirajparthan/Machine-Test/assets/100773790/68231631-7ea7-48f7-85dd-15984e58554f)

Then Created a 1 folders in the efs. The folder names is mysql. 

![Screenshot from 2024-02-17 10-24-39](https://github.com/abhirajparthan/Machine-Test/assets/100773790/c6b3233f-cdd7-48cc-abc1-9482bfa859bd)

( Here Master node keep as master, I will not initiate to deploy the containers in the master node. So I drain the IP from the cluster )

~~~
docker node update --availability drain ip-172-31-20-123
~~~

![Screenshot from 2024-02-17 10-20-49](https://github.com/abhirajparthan/Machine-Test/assets/100773790/6d3ce446-c8de-478d-82db-e7b2005a1381)

-------

For deploying the stack . I have created a folder flask_app in the manager node and created a docker-compose.yml file inside the folder. Here I am using mysql:5.6 image for database. Also I mounted the container /var/lib/mysql to efs folder( Here I am using bind mount. ).  Also I have aded the resource label of 2 nodes is flask and 1 is mysql
~~~
vi docker-compose.yml

---

version: '3.8'

services:

  database:
    image: mysql:5.6
    networks:
       - flask_net
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
    networks:
      flask_net:
    healthcheck:
      test: "mysqladmin ping -h 127.0.0.1 -u root --password=qwerty123! || exit 1"
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

networks:
  flask_net:
    driver: overlay
    ipam:
      driver: default
      config:
        - subnet: 172.16.0.0/24
~~~

I have deployed the stack using the command, My stack name in myflask
~~~
docker stack deploy -c docker-compose.yml myflask
~~~

After the deploying I have check the mysql replica using command 
~~~
docker service ls
~~~

![Screenshot from 2024-02-17 12-29-29](https://github.com/abhirajparthan/Machine-Test/assets/100773790/58825c2b-3ebe-4341-8a6c-db9d4d81c5fe)


Also I cloginto the container and checked the database and user name created depends on the env, Its created 

![Screenshot from 2024-02-17 12-14-28](https://github.com/abhirajparthan/Machine-Test/assets/100773790/3dc8540e-bd54-4d01-8149-9c451ce4181f)

Then checked the volume is mounted properly. Checked the mounted path /root/volume/mount/mysql I can see that the mounted files in the location.

![Screenshot from 2024-02-17 12-17-08](https://github.com/abhirajparthan/Machine-Test/assets/100773790/1b424e60-cf84-45bd-9af1-23eb80509153)

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
        'host': '172.31.27.71', #added Private IP of the server
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

For the database connectivity check I have imported table and table details to the mysql database. 

![Screenshot from 2024-02-17 14-04-24](https://github.com/abhirajparthan/Machine-Test/assets/100773790/f8b43784-46d8-4f99-ab30-34ca269f40b5)

Then modified the exsisting docker-compose.yml for the flask application

~~~
vi docker-compose.yml

---

version: '3.8'

services:

  database:    
    image: mysql:5.6    
    networks:
       - flask_net
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
    networks:
      flask_net:
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
    ipam:
      driver: default
      config:
        - subnet: 172.16.0.0/24


volumes:
  mysql_data:
  web_data:
~~~

After the I have deployed the flask application with mysql usig the command 

![Screenshot from 2024-02-17 21-57-49](https://github.com/abhirajparthan/Machine-Test/assets/100773790/7984e621-2170-416e-9b45-c00cadf5f989)

Then I have checked the services using the command 

![Screenshot from 2024-02-17 22-01-30](https://github.com/abhirajparthan/Machine-Test/assets/100773790/056e6726-e555-4625-ab4d-784b42b2e4b1)


Now the stack deployment is completed. We can acces the flask applicaton from the blow ip with port

~~~
http://3.14.6.225:5000/
http://3.144.211.214:5000/
~~~

We can see that the out put is below.
![Screenshot from 2024-02-17 22-00-07](https://github.com/abhirajparthan/Machine-Test/assets/100773790/5c55f21b-9ce1-4b9b-a9f0-924a0aa791de)

![Screenshot from 2024-02-17 22-00-14](https://github.com/abhirajparthan/Machine-Test/assets/100773790/1b2b2574-401a-48ff-9cef-0bd0bc4d8adf)





  
