# Machine-Test- Output with all steps and commands

###### 1st Stped
I have created a new 3 instance for configurung the docker and swarm, These instances are Ubuntu and the Version is 20.04. I have used 1 instance for Master and other 2 for worker nodes.

![Screenshot from 2024-02-16 17-04-59](https://github.com/abhirajparthan/Machine-Test/assets/100773790/d5cc96f2-ae98-4adf-aab5-ad85d2b23f26)


----

###### 2ns Step 

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

I have created the docker swarm cluster using the command 

~~~
docker swarm init
~~~

![Screenshot from 2024-02-16 17-44-51](https://github.com/abhirajparthan/Machine-Test/assets/100773790/2d8962d4-b10b-4293-97d4-70cb1e2086dc)






