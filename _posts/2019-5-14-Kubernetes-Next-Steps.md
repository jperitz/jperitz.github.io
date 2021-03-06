---
layout: post
title: Kubernetes: The Next Steps
---
Not long ago, I dipped my toes into the water with Kubernetes. I followed the tutorials, looked through the documentation, and learned the basics. You can read about my experience here. But much like swimming, having someone hold your hand and lead you into the water will only get you so far. Eventually, you have to let go and try to stay afloat on your own. That was my goal with this project: to create an app with Docker and deploy it with Kubernetes. 
	
After some research I decided to go with a Flask web app connected to a MySQL database. The web app itself isn’t that important, it’s just a vehicle to get more experience with Docker and Kubernetes, so if you’re following along feel free to use any web development framework you’re comfortable with. I decided that the server was important in order to get 2 containers that need to communicate and can be orchestrated together. 
	
I started by creating an Ubuntu VM with VMware Workstation since Docker-CE doesn’t play nicely with the the Windows 10 Home OS I have on my laptop, but that part is optional. Start by [installing Docker-CE](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository), [Docker-Compose](https://github.com/Yelp/docker-compose/blob/master/docs/install.md), [Minikube](https://medium.com/@naoko.reeves/how-to-install-minikube-on-ubuntu-bionic-18-04-af2d5435cc35) and [Kompose](https://kubernetes.io/docs/tasks/configure-pod-container/translate-compose-kubernetes/#kompose-convert). We’re going to be using app.py with Flask to handle routing for the web app (see below). 
```python
#/app/app.py
@app.route('/', methods=['GET', 'POST'])
def index():
    if request.method == "POST":
        details = request.form
        name = details['name']
        color = details['color']
        data_entry = (name, color)
        cnx = mysql.connector.connect(user='root', password='root', host='172.17.0.3', port='3306', database='colors')
        cursor = cnx.cursor()
        cursor.execute(add_entry, data_entry)
        cnx.commit()
        cursor.close()
        cnx.close()
        return 'Success!'
    return render_template('index.html')

@app.route('/api/data')
def colors() -> str:
    return json.dumps({'fav_colors': fav_colors()})
```
	
We’ll be serving an index.html page which allows users to enter their name and favorite color and send it to the database.The database itself is a simple sql file with a few pre-populated entries. We also need a requirements.txt page to help Docker install dependencies for the app. 
```SQL
#/db/colors.sql
CREATE DATABASE colors;
use colors;

CREATE TABLE fav_colors (
  name VARCHAR(20),
  color VARCHAR(10)
);

INSERT INTO fav_colors
  (name, color)
VALUES
  ('Cynthia', 'blue'),
  ('Jonathan', 'green'),
  ('Richard', 'yellow');
```

```
#/app/requirements.txt
Flask==0.12.3
mysql-connector
```

The only thing left to build the app is a Dockerfile which will install any dependencies and create a Docker image from it. [Docker Hub](https://hub.docker.com/) allows you to streamline the development process by creating an image repo and connecting it to your GitHub repo, which automatically updates the image every time changes are pushed to git. It’s a very handy feature that I strongly recommend for personal projects involving Docker.
```Dockerfile
#/app/Dockerfile
FROM ubuntu:18.04

LABEL maintainer="Jonathan Peritz <jperitz@ufl.edu>"

RUN apt-get update
RUN apt-get install -y python3 python3-pip python3-dev

# We copy just the requirements.txt first to leverage Docker cache
COPY ./requirements.txt /app/requirements.txt

WORKDIR /app

RUN pip3 install -r requirements.txt

COPY . /app

ENTRYPOINT [ "python3" ]

CMD [ "app.py" ]
```

We’re not done yet though. We need to use a docker-compose.yaml to make containers from the image we created earlier and an image from MySQL and connect the two. 
```Dockerfile
#docker-compose.yaml
version: "2"
services:
  app:
   # build: ./app
    image: jperitz/colors-2:latest
    links:
      - db
    ports:
      - "5000:5000"
    labels:
      kompose.service.type: nodeport

  db:
    image: mysql:5.7
    ports:
      - "32000:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - ./db:/docker-entrypoint-initdb.d/:ro
    labels:
      kompose.service.type: nodeport
```

The app container is built from the image created earlier,linked to the db container, with exposed port 5000. The db container is built from a MySQL image which mounts the sql file as a volume that docker can access. The purpose of the volume is persistent storage, which means that even if the database container is destroyed, the volume will still hold all the data, and you can just create another container. The docker-compose file was easier for me to write and understand than having to write several yaml files for kubernetes, so I used kompose to convert to a format kubernetes prefers. I ran the following command to convert from docker-compose.yaml to four new .yaml files: 
```
kompose --file docker-compose.yaml comvert --volumes hostPath
```

Now we’re ready to roll! Execute the following commands to run the app:
```
Minikube start #note: I used sudo -E minikube start --vm-driver=none, but you should only consider doing that if you’re running these commands inside a VM.
Kubectl apply -f app-service.yaml,db-service.yaml,app-deployment.yaml,db-deployment.yaml
```

Now everything is deployed and you can see that everything is running smoothly with kubectl get deployment,svc,pods. Use kubectl describe services to see the ip address of the app endpoint. Connect to that address on port 5000 and you should see the app running. 

![form submission]({{ site.baseurl }}/images/css_added.png "form submission")

![sucessful submission]({{ site.baseurl }}/images/kubernetes_success.png "successful submission")

![filled database]({{ site.baseurl }}/images/kubernetes_filled_db.png "filled database")

Use the address $IP:5000/api/data to get a json dump of the database. You can play around with kubectl commands and see how it affects the app. One I found particularly useful in development was the following:
```
Kubectl get rs
Kubectl scale --replicas=0 rs/”appname”
```

That scales down the number of replicas of the app to 0. Kubernetes will drive the app back to the desired state of 1 replica, but first it will pull the latest image of the app from Docker. When I ran into bugs I just changed the code, pushed the changes to git, and then a few minutes later when the Docker image updated, I ran those commands to pull the latest image. That way, I didn’t have to shut down the deployment and restart every time I wanted to change something.

There you have it! A simple locally hosted web app with docker and kubernetes that can serve as a template for small projects. The app itself is minimalist and not very attractive, but changing that is just a matter of adding more html, css and changing app.py to route to more pages. 

Source available here: <https://github.com/jperitz/Kubernetes_Project2>
