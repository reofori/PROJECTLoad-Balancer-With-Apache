# Load-balancer-with-Apache

* After completing DevOps tooling website solution Project, we might wonder how a user will be accessing each of the webservers using 3 different IP addreses or 3 different DNS names.
we might also wonder what is the point of having 3 different servers doing exactly the same thing. 

* When we access a website in the Internet we use an URL and we do not really know how many servers are out there serving our requests, this complexity is hidden from a regular user, but in case of websites that are being visited by millions of users per day (like Google or Reddit) it is impossible to serve all the users from a single Web Server (it is also applicable to databases, but for now we will not focus on distributed DBs).

* Each URL contains a domain name part, which is translated (resolved) to IP address of a target server that will serve requests when open a website in the Internet. Translation (resolution) of domain names is perormed by DNS servers, the most commonly used one has a public IP address 8.8.8.8 and belongs to Google. You can try to query it with nslookup command:

```
nslookup   8.8.8.8
Server:   UnKnownAddress: 103.86.99.99

Name:   dns.google
Address: 8.8.8.8
```

* When you have just one Web server and load increases - you want to serve more and more customers, you can add more CPU and RAM or completely replace the server with a more powerful one - this is called "vertical scaling". This approach has limitations - at some point you reach the maximum capacity of CPU and RAM that can be installed into your server.

* Another approach used to cater for increased traffic is "horizontal scaling" - distributing load across multiple Web servers. This approach is much more common and can be applied almost seamlessly and almost infinitely (you can imagine how many server Google has to serve billions of search requests).

* Horizontal scaling allows to adapt to current load by adding (scale out) or removing (scale in) Web servers. Adjustment of number of servers can be done manually or automatically (for example, based on some monitored metrics like CPU and Memory load).Property of a system (in our case it is Web tier) to be able to handle growing load by adding resources, is called "Scalability".In our set up in Project-7 we had 3 Web Servers and each of them had its own public IP address and public DNS name.

* A client has to access them by using different URLs, which is not a nice user experience to remember addresses/names of even 3 server, let alone millions of Google servers.In order to hide all this complexity and to have a single point of access with a single public IP address/name, a Load Balancer can be used. A Load Balancer (LB) distributes clients' requests among underlying Web Servers and makes sure that the load is distributed in an optimal way.

* Property of a system (in our case it is Web tier) to be able to handle growing load by adding resources, is called "Scalability".

* In our set up in Project-7 we had 3 Web Servers and each of them had its own public IP address and public DNS name. A client has to access them by using different URLs, which is not a nice user experience to remember addresses/names of even 3 server, let alone millions of Google servers.

* In order to hide all this complexity and to have a single point of access with a single public IP address/name, a Load Balancer can be used. A Load Balancer (LB) distributes clients' requests among underlying Web Servers and makes sure that the load is distributed in an optimal way.

Let us take a look at the updated solution architecture with an LB added on top of Web Servers (for simplicity let us assume it is a software L7 Application LB, for example - Apache, NGINX or HAProxy).


## Objective

   ![image 1](images/image%201.jpg)

In this project, we will enhance our Tooling Website solution by adding an Apache Load Balancer to distribute traffic between two Web Servers. This will allow users to access our website using a single URL.

## Task
Deploy and configure an Apache Load Balancer on a separate Ubuntu EC2 instance, ensuring that users can be served by both Web Servers through the Load Balancer.

For simplicity, this solution will implement load balancing with 2 Web Servers, but the approach can be extended to accommodate more servers.

## Prerequisites

Ensure you have the following servers already installed and configured from the previous project:

1. Two RHEL8 Web Servers
2. One MySQL DB Server (Ubuntu 24.04)
3. One RHEL8 NFS Server

   ![image 1](images/image%202.jpg)




## Step 1 - Configure Apache as a Load Balancer

1. Create an Ubuntu 20.04 EC2 instance and name it Project-8-apache-lb, and open TCP port 80 by creating an inbound rule in the security group for this instance.

    ![image 1](images/image%203.jpg)


2. Install Apache and Configure Load Balancer

   ```
   sudo apt update
   ```
   
3. Install Apache and required modules:

   ```
   sudo apt install apache2 -y
   sudo apt-get install libxml2-dev
   ```
   
4. Enable the necessary Apache modules:

   ```
   sudo a2enmod rewrite
   sudo a2enmod proxy
   sudo a2enmod proxy_balancer
   sudo a2enmod proxy_http
   sudo a2enmod headers
   sudo a2enmod lbmethod_bytraffic
   ```

5. Restart Apache to apply the changes:

   ```
   sudo systemctl restart apache2
   ```

## Step 2 - Configure Load Balancing

1. Open the Apache configuration file:

   ```
   sudo vi /etc/apache2/sites-available/000-default.conf
   ```

2. Add the following configuration within the <VirtualHost *:80> section:

   ```
   <VirtualHost *:80>
    <Proxy "balancer://mycluster">
        BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=1
        BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=1
        ProxySet lbmethod=bytraffic
        # ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
   </VirtualHost>
   ```
   Replace and with the private IPs of your two RHEL8 web servers.
   
    ![image 1](images/image%204.jpg)


4. Restart Apache to Apply Changes:

   ```
   sudo systemctl restart apache2
   ```

## Step 3 - Test the Load Balancer and Verify Logs on the Web Servers 

1. Access the public IP of your Project-8-apache-lb instance in a browser:

   ```
   http://<Load-Balancer-Public-IP>/index.php
   ```

 ![image 1](images/image%205.jpg)


2. Open two SSH terminals, one for each Web Server, and run the following command to monitor the logs:

   ```
   sudo tail -f /var/log/httpd/access_log
   ```

3. Refresh the browser several times and confirm that both Web Servers are receiving HTTP GET requests from the Load Balancer. The logs should indicate that traffic is being distributed evenly between the servers.


## Step 3 - Configure Local DNS Names (Optional)

 1. Open the /etc/hosts file on the Load Balancer server:

    ```
    sudo vi /etc/hosts
    ```

2. Add entries for the Web Servers:

   ```
   <WebServer1-Private-IP-Address> Web1
   <WebServer2-Private-IP-Address> Web2
   ```

3. Save the file and exit the editor.

4. Update your Apache configuration file ```/etc/apache2/sites-available/000-default.conf``` to use the new names instead of IP addresses:

   ```
   <Proxy "balancer://mycluster">
    BalancerMember http://Web1:80 loadfactor=5 timeout=1
    BalancerMember http://Web2:80 loadfactor=5 timeout=1
    ProxySet lbmethod=bytraffic
   </Proxy>
   ```

5. Save and exit the file, then restart Apache:

   ```
   sudo systemctl restart apache2
   ```

   ![image 1](images/image%205.jpg)

  You can also test if the names resolve correctly from the Load Balancer by using curl:

   ![image 1](images/image%206.jpg)


# Tagrget Architecture

At this stage, your setup should look like this:

1. Apache Load Balancer (Ubuntu 24.04 EC2 instance)

2. Two Web Servers (RHEL8 EC2 instances)

3. One MySQL DB Server (Ubuntu 24.04 EC2 instance)

4. One RHEL8 NFS Server (RHEL8 EC2 instance)

   ![image 1](images/image%208.jpg)


   ## CONGRATULATIONS!!!!!!! YOU HAVE JUST IMPLEMENTED A LOAD BALANCER SOLUTION WITH APACHE FOR YOUR DEVOPS TEAM.





























