# Machine-Test- Output with all steps and commands

###### 1st Step
I have created a new 3 instance in AWS account for configurung the docker and swarm, These instances are Ubuntu and the Version is 20.04. I have used 1 instance for Master and other 2 for worker nodes.

![Screenshot from 2024-02-16 17-04-59](https://github.com/abhirajparthan/Machine-Test/assets/100773790/d5cc96f2-ae98-4adf-aab5-ad85d2b23f26)


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

![Screenshot from 2024-02-16 18-00-30](https://github.com/abhirajparthan/Machine-Test/assets/100773790/f22c950b-d59b-4a23-8e5a-1faaf0c8c0a4)


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

Then Created a 2 folders in the efs. The folder names are mysql and web. 

![Screenshot from 2024-02-17 10-24-39](https://github.com/abhirajparthan/Machine-Test/assets/100773790/c6b3233f-cdd7-48cc-abc1-9482bfa859bd)

( Here Master node keep as master, I will not initiate to deploy the containers in the master node. So I drain the IP from the cluster )

~~~
docker node update --availability drain ip-172-31-20-123
~~~

![Screenshot from 2024-02-17 10-20-49](https://github.com/abhirajparthan/Machine-Test/assets/100773790/6d3ce446-c8de-478d-82db-e7b2005a1381)

-------

For deploying the stack . I have created a folder flask_app in the manager node and created a docker-compose.yml file inside the folder. Here I am using mysql:5.6 image for database. Also I mounted the container /var/lib/mysql to efs folder( Here I am using bind mount. ). 
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
    deploy:
      replicas: 1



networks:
  flask_net:
    driver: overlay

volumes:
  mysql_data:
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
        'host': 'database',
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






  
