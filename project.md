## LOAD BALANCER SOLUTION WITH APACHE
After reading about different [Load Balacing concepts](https://www.nginx.com/resources/glossary/load-balancing/) and difference between [l4 Network LB](https://www.nginx.com/resources/glossary/layer-4-load-balancing/) and [l7 Application LB](https://www.nginx.com/resources/glossary/layer-7-load-balancing/) 

having  a look at an updated solution architecture with an LB added on top of Web Servers (for simplicity let us assume it is a software L7 Application LB, for example – [Apache](https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html), [NGINX](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/) or [HAProxy](http://www.haproxy.org/)

![](/Tooling-Website-Infrastructure-wLB.png)

In this project we will enhance our **Tooling Website** solution by adding a Load Balancer to disctribute traffic between Web Servers and allow users to access our website using a single URL.

![](/prerequisites-project8.png)

### CONFIGURE APACHE AS A LOAD BALANCER
1. Create an Ubuntu Server 20.04 EC2 instance and name it Project-8-apache-lb, so your EC2 list will look like this:

![](/project%208.PNG)

2. Open TCP port 80 on Project-8-apache-lb by creating an Inbound Rule in Security Group.

3. Install Apache Load Balancer on Project-8-apache-lb server and configure it to point traffic coming to LB to both Web Servers:


```#Install apache2



sudo apt update



sudo apt install apache2 -y


sudo apt-get install libxml2-dev



#Enable following modules: 


sudo a2enmod rewrite

sudo a2enmod proxy


sudo a2enmod proxy_balancer


sudo a2enmod proxy_http

sudo a2enmod headers

sudo a2enmod lbmethod_bytraffic


#Restart apache2 service

sudo systemctl restart apache
```
make sure apache2 is up and running 

`sudo systemctl status apache2`

configure load balacing 

```sudo vi /etc/apache2/sites-available/000-default.conf

#Add this configuration into this section <VirtualHost *:80>  </VirtualHost>

<Proxy "balancer://mycluster">
               BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=1
               BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=1
               ProxySet lbmethod=bytraffic
               # ProxySet lbmethod=byrequests
        </Proxy>

        ProxyPreserveHost On
        ProxyPass / balancer://mycluster/
        ProxyPassReverse / balancer://mycluster/

#Restart apache server

sudo systemctl restart apache2
```
[bytraffic](https://httpd.apache.org/docs/2.4/mod/mod_lbmethod_bytraffic.html) balancing method will distribute incoming load between your Web Servers according to current traffic load. We can control in which proportion the traffic must be distributed by loadfactor parameter.
we also have other methods, like: [bybusyness](https://httpd.apache.org/docs/2.4/mod/mod_lbmethod_bybusyness.html), [byrequests](https://httpd.apache.org/docs/2.4/mod/mod_lbmethod_byrequests.html), [heartbeat](https://httpd.apache.org/docs/2.4/mod/mod_lbmethod_heartbeat.html)

4. Verify that our configuration works – try to access your LB’s public IP address or Public DNS name from your browser:


`http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php`

![](/project%208.1.PNG)



**Note:** If in the Project-7 you mounted */var/log/httpd/* from your Web Servers to the NFS server – unmount them and make sure that each Web Server has its own log directory.

`sudo umount -f /var/log/httpd`

Open two ssh/Putty consoles for both Web Servers and run following command:

`sudo tail -f /var/log/httpd/access_log`


![](/apache%20curl.PNG)

Try to refresh your browser page http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php several times and make sure that both servers receive HTTP GET requests from your LB – new records must appear in each server’s log file. The number of requests to each server will be approximately the same since we set loadfactor to the same value for both servers – it means that traffic will be disctributed evenly between them.

If you have configured everything correctly – your users will not even notice that their requests are served by more than one server.

**side self study**
Read more about different configuration aspects of [Apache mod_proxy_balancer module](https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html). Understand what sticky session means and when it is used.

#### Optional Step – Configure Local DNS Names Resolution

Sometimes it is tedious to remember and switch between IP addresses, especially if you have a lot of servers under your management.
What we can do, is to configure local domain name resolution. The easiest way is to use /etc/hosts file, although this approach is not very scalable, but it is very easy to configure and shows the concept well. So let us configure IP address to domain name mapping for our LB.

```
#Open this file on your LB server

sudo vi /etc/hosts

#Add 2 records into this file with Local IP address and arbitrary name for both of your Web Servers

<WebServer1-Private-IP-Address> Web1
<WebServer2-Private-IP-Address> Web2
```

Now you can update your LB config file with those names instead of IP addresses.
```
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
```
You can try to curl your Web Servers from LB locally curl http://Web1 or curl http://Web2 – it shall work.
![](/curl1.PNG)


Remember, this is only internal configuration and it is also local to your LB server, these names will neither be ‘resolvable’ from other servers internally nor from the Internet.


**Targed Architecture**

Now your set up looks like this:

![](/project8_final.png)

