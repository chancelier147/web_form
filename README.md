**TOOL OF THE TRADE PART 2**

Build a simple LAMP stack based web form and move to Kubernetes

(Submit relevant Docker and YAML files for the project along with project folder)

This project is subdivided into two steps, the first step will be to build our docker images and then deploy them on a kubernetes cluster.

**1. Docker**

We are going to dockerise our project, it's a simple web form. Here is the content of the project :
```
src/
├── errors.php
├── index.php
├── login.php
├── register.php
├── server.php
└── style.css
```
We will add the necessary files for Dockerization and deployment.

We create a new directory named &quot;form-app-docker&quot;, and in this directory we put the project folder called &quot;src&quot;. Also we add the following files: Dockerfile, 000-default.conf and start-apache.

The file &quot;000-default.conf&quot; is used to configure a virtualhost here is its content :

```
<VirtualHost *:80>
  ServerAdmin webmaster@localhost
  DocumentRoot /var/www/html

  <Directory /var/www>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
</VirtualHost>
```
The "start-apache" file, manages the port and the start of apache

its contents:
```
#!/usr/bin/env bash
sed -i "s/Listen 80/Listen ${PORT:-80}/g" /etc/apache2/ports.conf
sed -i "s/:80/:${PORT:-80}/g" /etc/apache2/sites-enabled/*
apache2-foreground
```

And ensure the file is executable :  chmod 755 start-apache

And finally the "Dockerfile" which allows to dockerise our application

its contents :
```
FROM php:7-apache
MAINTAINER tizieemmanuel@hotmail.fr

RUN docker-php-ext-install mysqli

COPY 000-default.conf /etc/apache2/sites-available/000-default.conf
COPY start-apache /usr/local/bin
RUN a2enmod rewrite

# Copy application source
COPY src /var/www/html
RUN chown -R www-data:www-data /var/www/html

CMD ["start-apache"]
```

At this point we have this structure :
```
form-app-docker/
├── 000-default.conf
├── Dockerfile
├── src
│   ├── errors.php
│   ├── index.php
│   ├── login.php
│   ├── register.php
│   ├── server.php
│   └── style.css
└── start-apache
```
We can now start building our Docker image.

To build, we execute the following command at the root of the folder :
```
docker build -t form-app .
```
We get the following image: **form-app:latest**

Our web form has been dockerised, but it needs to communicate with a database, so we will run a mysql database. Preferably we will use mariadb.

To do this we won&#39;t use a Dockerfile, but rather the official mariadb image available on Docker Hub.

We execute the following command :
```
docker run --name registration-db -e MYSQL_ROOT_PASSWORD=root -v /data:/var/lib/mysql -d mariadb:10.3-bionic
```

In this command we use the -v option to link a volume to our mariadb container in order to save the database data.

We also start the web form container with the following command :
```
docker run -d -p 6868:80 --name my-form-app form-app:latest
```

For the communication between the web form container and the database container we will create a bridge type network. Then insert the two containers in the same private network.
```
docker network create -d bridge manu-bridge
docker network connect manu-bridge registration-db
docker network connect manu-bridge my-form-app
```

To test our application we will connect to the database server, create the database and a table.

We log in with the following command:
```
docker exec -it d2f60c3f4b95 mysql -h172.17.0.9 -uroot -p
```
```
>CREATE DATABASE registration;

>use registration

>CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `username` varchar(100) NOT NULL,
  `email` varchar(100) NOT NULL,
  `password` varchar(100) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

**d2f60c3f4b95** : The container ID

**172.17.0.9** : The container IP address

we can access the application via the browser by entering the server&#39;s ip address and port 6868. [http://ip\_server:6868]



**2. Kubernetes**

Now we&#39;re going to move on kubernetes.

We will deploy in a namespace &quot;development&quot;

**First step : deploy a database server like mysql or mariadb**
**Second step : deploy our form web app on kubernetes**

**2.1. Deploy mariadb**

In order to deploy mariadb, we will use two files, the first file for persistent volume and the second file for deployment.

mariadb-pv.yaml : persistent volume and persistent volume claim

mariadb-deployment.yaml : deployment and service

here are the contents of the files

**(mariadb-pv.yaml)**
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

**(mariadb-deployment.yaml)**
```
apiVersion: v1
kind: Service
metadata:
  name: registration-db
spec:
  ports:
  - port: 3306
  selector:
    app: registration-db
  type: LoadBalancer
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: registration-db
spec:
  selector:
    matchLabels:
      app: registration-db
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: registration-db
    spec:
      containers:
      - image: mariadb:10.3-bionic
        name: registration-db
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-pass
                key: password
        ports:
        - containerPort: 3306
          name: registration-db
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```

To deploy we execute the following actions :

_a- creation of a secret to store the root password_
```
kubectl create secret generic mysql-pass --from-literal=password=root -n development
```
_b- deploy the pv and pvc_
```
$kubectl apply -f mariadb-pv.yaml -n development
```
_c- deploy mariadb and its service_
```
kubectl apply -f mariadb-deployment.yaml -n development
```

**2.2. Create a database and a table**

_a- enter in the pod_
```
kubectl exec -it registration-db-f9bdb4df6-8nvd2 bash -n development
```
_b- create the database and table_
```
>CREATE DATABASE registration;

>use registration

>CREATE TABLE `users` (
  `id` int(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `username` varchar(100) NOT NULL,
  `email` varchar(100) NOT NULL,
  `password` varchar(100) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```
**2.3. Deploy the form web app**

To deploy our docker image to kubernetes, we pushed the image to docker hub and then used it in our deployment file. this file contains the deployment and the service file

here are its contents

**(form-deployment.yaml)**
```
apiVersion: v1
kind: Service
metadata:
  name: form-app
spec:
  ports:
  - port: 80
  selector:
    app: form-app
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: form-app
  labels:
    app: form-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: form-app
  template:
    metadata:
      labels:
        app: form-app
    spec:
      containers:
      - name: form
        image: emmanuel147/form-app:latest
        ports:
        - containerPort: 80
```
We can deploy with the following command :
```
kubectl apply -f form-deployment.yaml -n development
```
We will retrieve the ip address of our database server and then use it in the pod of the web form.

_a- Retrieve the database server ip_
```
kubectl describe pod registration-db-f9bdb4df6-8nvd2 -n development
```

Status:       Running
IP:           10.244.2.193

_b- Set the ip address_

Get the right file to set the ip address of the database server
```
kubectl exec form-app-5fd88df9bb-fhdqm -n development -it -- cat /var/www/html/server.php > server.php
```

set the right ip address in the server.php file and push in the pod
```
kubectl cp server.php development/form-app-5fd88df9bb-5c62q:/var/www/html/server.php
```
**2.4. Finally, we retrieve the ip address of our application with the following command**
```
kubectl get service -n development
```

output :
```
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
form-app          LoadBalancer   10.96.49.108    192.168.9.104   80:31458/TCP     28m
```

We access the application with the external ip address [http://192.168.9.104] in my case
