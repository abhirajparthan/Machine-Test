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

( Here Master node keep as master, I will not initiate to deploy the containers in the master node. So I drain the IP from the cluster )

~~~
docker node update --availability drain ip-172-31-20-123
~~~

![Screenshot from 2024-02-17 10-20-49](https://github.com/abhirajparthan/Machine-Test/assets/100773790/6d3ce446-c8de-478d-82db-e7b2005a1381)










  
