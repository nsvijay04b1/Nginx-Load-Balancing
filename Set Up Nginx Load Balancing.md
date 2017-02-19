# Set Up Nginx Load Balancing
![A](https://github.com/nu11secur1ty/Nginx-Load-Balancing/blob/master/photo/load-balancer-graph-1-1024x522.png)



# About Load Balancing

Load balancing is a useful mechanism to distribute incoming traffic around several capable Virtual Private servers.By apportioning the processing mechanism to several machines, redundancy is provided to the application -- ensuring fault tolerance and heightened stability. The Round Robin algorithm for load balancing sends visitors to one of a set of IPs. At its most basic level Round Robin, which is fairly easy to implement, distributes server load without implementing considering more nuanced factors like server response time and the visitors’ geographic region. Load balancing is an excellent way to scale out your application and increase it’s performance and redundancy. Nginx, which is a popular web server software, can be configured as a simple yet powerful load balancer to improve your servers resource availability and efficiency. In a load balancing configuration nginx acts as single entrance point to a distributed web application working on multiple separate servers.

![A](https://github.com/nu11secur1ty/Nginx-Load-Balancing/blob/master/photo/load-balancer.png)

This guide describes how to set up load balancing with nginx for your cloud servers. As a prerequisite you’ll need to have at least two hosts with a web server software installed and configured to see the benefit of the load balancer. If you already have one web host set up, you can use the Server Cloning feature available at your UpCloud Control Panel.

# Installing nginx

The first thing to do is to set up a new host that will serve as your load balancer. Go ahead and deploy a new instance at your UpCloud Control Panel if you haven’t already. Currently nginx packages are available on the latest versions of CentOS, Debian and Ubuntu, so pick which ever of these you prefer.

After setting up the server the way you like, adding users, running updates, and so on, install the latest stable nginx using one of the following methods.

# CentOS

```
sudo vim /etc/yum.repos.d/nginx.repo

. Enter the following to the file
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1

. Save and exit, then use the command below
sudo yum update
sudo yum install nginx
```

# Debian

```
sudo nano /etc/apt/sources.list

. Add the following to the end of the list
deb http://nginx.org/packages/debian/ jessie nginx
deb-src http://nginx.org/packages/debian/ jessie nginx

. Save the file, exit the editor and then run the following commands
sudo aptitude update
sudo aptitude install nginx
```
# Ubuntu

```
nginx=stable
sudo add-apt-repository ppa:nginx/$nginx
sudo apt-get update
sudo apt-get install nginx
```

Once installed change directory into the nginx main configuration folder.

```
cd /etc/nginx/
```

Now depending on your OS the web server configuration files will be in one of two places.

Ubuntu and Debian follow a rule for storing virtual host files in /etc/nginx/sites-available/, which are enabled through symbolic links to /etc/nginx/sites-enabled/. You can use the command below to enable any new virtual host files.


```
sudo ln -s /etc/nginx/sites-available/<vhost> /etc/nginx/sites-enabled/<vhost>
```

CentOS users can find their host configuration files under /etc/nginx/conf.d/ in which any .conf -type virtual host file gets loaded.

Check that you can find at least the default configuration and then restart nginx.


```
sudo service nginx restart
```

Test that the server replies to HTTP requests by opening the load balancer server’s public IP address in your web browser. When you see the default welcoming page for nginx the installation was successful.


![A](https://github.com/nu11secur1ty/Nginx-Load-Balancing/blob/master/photo/nginx-welcome-page.png)


If you are having trouble loading the page, check that a firewall is not blocking your connection. For example on CentOS 7 the default firewall rules do not allow HTTP traffic, enable it with the commands below.


```
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```


Then try reloading your browser.

Configuring nginx as a load balancer

With nginx installed and tested you can start configuring it for load balancing. In essence all you need to do is setup nginx with instructions for which type of connections to listen to and where to redirect them. To accomplish this, create a new configuration file using which ever text editor you prefer, for example with nano:

```
sudo nano /etc/nginx/conf.d/load-balancer.conf
```


In the load-balancer.conf you’ll need to define the following two segments, upstream and 
server, see the examples below.


```
# Define which servers to include in the load balancing scheme. 
# It's best to use the servers' private IPs for better performance and security.
# You can find the private IPs at your UpCloud Control Panel Network section.

upstream backend {
   server 10.1.0.101; 
   server 10.1.0.102;
   server 10.1.0.103;
}

# This server accepts all traffic to port 80 and passes it to the upstream. 
# Notice that the upstream name and the proxy_pass need to match.

server {
   listen 80; 

   location / {
      proxy_pass http://backend;
   }
}
```

Then save the file and exit the editor.

Next you’ll need to disable the default server configuration you earlier tested was working after the installation. Again depending on your OS this part differs slightly.

On Debian and Ubuntu systems you’ll need to remove the default symbolic link from the sites-enabled folder.

```
sudo rm /etc/nginx/sites-enabled/default
```

CentOS hosts don’t use the same linking, instead simply rename the default.conf in the conf.d/ directory to something that doesn’t end with .conf, for example:

```
sudo mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.disabled
```


Then use the following to restart nginx.


```
sudo service nginx restart
```


Check that nginx starts successfully. If the restart fails, take a look at the  /etc/nginx/conf.d/load-balancer.conf you just created to make sure there are no mistypes or missing semicolons.

You should now be passed to one of your back-end servers when entering the load balancer’s public IP address in your web browser.
Load balancing methods

Load balancing with nginx uses round-robin algorithm by default, if no other method is defined, like in the first example above. With round-robin scheme each server is selected in turns according to the order you set them in the load-balancer.conf -file. This balances the number of requests equally for short operations.

Least connections based load balancing is an other straight forward method. As the name suggests, this method directs the requests to the server with the least active connections at that time. It works more fairly than round-robin would with applications where requests might sometimes take longer to complete.

To enable least connections balancing method add the parameter least_conn to your upstream -section as shown in the example below.


```
upstream backend {
   least_conn;
   server 10.1.0.101; 
   server 10.1.0.102;
   server 10.1.0.103;
}
```

While round-robin and least connections balancing schemes are fair and have their uses, they however cannot provide session persistence. If your web application requires that the users are subsequently directed to the same back-end server as during their previous connection, you should use IP hashing method instead. IP hashing uses the visitors IP address as a key to determine which host should be selected to server the request. This allows the visitors to be each time directed to the same server, granted that the server is available and the visitor’s IP address hasn’t changed.

To use this method, add the ip_hash -parameter to your upstream -segment like in the example underneath.

```
upstream backend {
   ip_hash;
   server 10.1.0.101; 
   server 10.1.0.102;
   server 10.1.0.103;
}
```

In a server setup where the available resources between different hosts are not equal it might be desirable to favour some servers over others. Defining server weights allows you to further fine tune load balancing with nginx. The server with the highest weight in the load balancer is selected the most often.


```
upstream backend {
   server 10.1.0.101 weight=4; 
   server 10.1.0.102 weight=2;
   server 10.1.0.103;
}
```

For example in the configuration shown above the first server is selected twice as often as the second, which again gets twice the requests compared to the third.

# Load balancing with HTTPS enabled

Enabling HTTPS for your site is a great way to protect your visitors and their data. If you haven’t yet implemented encryption on your web hosts, we highly recommend taking a look at our guide for How to Install Let’s Encrypt on Nginx.

Using encryption with a load balancer is easier than you might think. All you need to do is add an other server section to your load balancer configuration file which listens to HTTPS traffic at port 443 with SSL and set up a proxy_pass to your upstream segment like with the HTTP in the previous example above.

Open your configuration file again for edit.

```
sudo nano /etc/nginx/conf.d/load-balancer.conf
```

Then add the following server segment to the end of the file.

```
server {
   listen 443 ssl;
   server_name <domain name>;
   ssl_certificate /etc/letsencrypt/live/<domain name>/cert.pem;
   ssl_certificate_key /etc/letsencrypt/live/<domain name>/privkey.pem;

   location / {
      proxy_pass http://backend;
   }
}
```

Then save the file, exit the editor and restart nginx again with


```
sudo service nginx restart
```
Setting up encryption at your load balancer while using the private network connections to your back-end has some great advantages.

   . As only your UpCloud servers have access to your private network, it allows you to terminate the SSL at the load balancer and thus only passing forward HTTP connections.
   . It also greatly simplifies your certificate management as you can obtain and renew the certificates from a single host.

With the HTTPS enabled you also have the option to enforce encryption to all connections to your load balancer. Simply update your server segment listening to port 80 with a server name and a redirection to your HTTPS port, then remove or comment out the location portion as it’s no longer needed. See the example below.


```
server {
   listen 80;
   server_name <domain name>;
   return 301 https://$server_name$request_uri;

   #location / {
   #   proxy_pass http://backend;
   #}
}
```

Save the file again after making the changes and then restart nginx.

```
sudo service nginx restart
```



Now all connections to your load balancer will be served over encrypted HTTPS connection and requests to the unencrypted HTTP will be redirected to use HTTPS as well. This provides a seamless transition into encryption with nothing required from your visitors.

# Health checks

In order to know which servers are available nginx’s implementations of reverse proxy includes passive server health checks. If a server fails to respond to a request or replies with an error, nginx will note the server as failed and will try to avoid directing connections to that server for a time.

The number of consecutive unsuccessful connection attempts within a certain time period can be defined in the load balancer configuration file by setting a parameter max_fails to the server lines. By default, when no max_fails is specified, this value is set to 1. Optionally setting the max_fails to 0 will disable health checks to that server.

If max_fails is set to a value greater than 1 the subsequent fails must happen within a specific time frame for the fails to count. This time frame is specified by a parameter fail_timeout, which also defines how long the server should be considered failed. By default the fail_timeout is set to 10 seconds.

After a server is marked failed and the time set by fail_timeout has passed, nginx will begin to gracefully probe the server with client requests. If the probes return successful, the server is again marked live and included in the load balancing as normal.

```
upstream backend {
   server 10.1.0.101 weight=5;
   server 10.1.0.102 max_fails=3 fail_timeout=30s;
   server 10.1.0.103;
}
```


Using the health checks allows you to adapt your server back-end to the current demand by powering up or down hosts as required. Starting up additional servers during high traffic can easily increase your application performance when new resources become automatically available to your load balancer.

# Conclusions

If you wish to improve your web application performance and availability, setting up a load balancer is definitely something to consider. Load balancing with nginx is powerful yet relatively simple to setup and together with an easy encryption solution, such as Let’s Encrypt client, it makes for a great front-end to your web farm. Check out the documentation for upstream over at nginx.org to learn more.

While using multiple hosts protects your web service with redundancy, the load balancer itself can still leave a single point of failure. You can further improve high availability by setting up a floating IP between multiple load balancers. You can find out more at our article for Floating IPs on UpCloud.


# Have fun with nu11secur1ty =)
























































































































































































