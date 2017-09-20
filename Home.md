This is an entry level tutorial for engineers who wish to build a scalable web application but do not know how to get started. The demo application takes advantage of the various services offered by AWS such as Elastic Computer Cloud (EC2), Elastic Load Balancer (ELB), Relational Database Service (RDS), ElastiCache, Simple Storage Service (S3), Identity and Access Management (IAM), as well as CloudWatch and AutoScaling. By using AWS we are making things easier so that an engineer with a little experience with Linux can easily finish this tutorual in a couple of hours. If you are building a scalable web application on your own infrastructure or on another public cloud, the theory should be the same but the actual implementation might be somewhat different.

Please note that  the code provided here is for demo only and is not intended for production usage.

**(1) Executive Summary**

![Web Demo](http://www.qyjohn.net/wp-content/uploads/2015/01/web-demo.png)

I want to build a scalable web application that can serve a large amount of users. I really don’t know how many users I will get so the technical specification is as many as possible. As shown in the above design, this is a photo sharing website with the following functions:

(1) User authentication (login / logout).

(2) Logined users can upload photos.

(3) The default page displays the latest N uploaded photos.

In this tutorial, we will accomplish this goal through the following five levels:

(0) A basic version with all the components deployed on one single server. The web application is developed with PHP, using Apache as the web serer, with MySQL as the database to store user upload information.

(1) Based on the basic version we developed in level (0), scale the application to two or more servers.

(2) Use S3 to provide a shared storage for user uploads.

(3) Dynamically scale the size of your server fleet according to the actual traffic to your web application.

(4) Implement a cache layer between the web server and the database server.

**(2) LEVEL 0**

![LEVEL 0](http://www.qyjohn.net/wp-content/uploads/2017/03/Slide3.png)

In this level, we will build a basic version with all the components deployed on one single server. The web application is developed with PHP, using Apache as the web serer, with MySQL as the database to store user upload information. You do not need to write any code, because I have a set of demo code prepared for you. You just need to launch an EC2 instance, carry out some basic configurations, then deploy the demo code.

Login to your AWS Console and navigate to the EC2 Console. Launch an EC2 instance with a Ubuntu 14.04 or Ubuntu 16.04 AMI. Make sure that you allocate a public IP to your EC2 instance. In your security group settings, open port 80 for HTTP and port 22 for SSH access. After the instance becomes “running” and passes health checks, SSH into your EC2 instance to setup software dependencies and download the demo code from Github to the default web server folder:

If you are using Ubuntu 14.04, install the following software:

~~~~
$ sudo apt-get update
$ sudo apt-get install git mysql-server
$ sudo apt-get install apache2 php5 php5-mysql php5-curl php5-memcached
~~~~

If you are using Ubuntu 16.04, the software installation part is a little bit different:

~~~~
$ sudo apt-get update
$ sudo apt-get install git mysql-server
$ sudo apt-get install apache2 php libapache2-mod-php php-mcrypt php-mysql php-curl php-xml php-memcached
~~~~

You might need to restart your Apache web server:

~~~~
$ sudo service apache2 restart
~~~~

Then we clone the git repository:

~~~~
$ cd /var
$ sudo chown -R ubuntu:ubuntu www
$ cd /var/www/html
$ git clone https://github.com/qyjohn/web-demo
~~~~

Then we create a MySQL database and a MySQL user for our demo. Here we use “web_demo” as the database name, and “username” as the MySQL user.

~~~~
$ mysql -u root -p
mysql> CREATE DATABASE web_demo;
mysql> CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON web_demo.* TO 'username'@'localhost';
mysql> quit
~~~~

In the code you clone from Github, we have pre-populated some demo data as examples. We use the following command to import the demo data in web_demo.sql to the web_demo database:

~~~~
$ cd /var/www/html/web-demo
$ mysql -u username -p web_demo < web_demo.sql
~~~~

The LEVEL 0 demo code is implemented in a two PHP files index.php and config.php. Before we can make it work, there are some minor modifications needed:

(1) Use a text editor to open config.php, then change the username and password for your MySQL installation. The example commands before created a DB user named "username" with the password "password". This is the default DB credentials configured in config.php. If you used a different username or password, update config.php accordingly.

(2) Change the ownership of folder “uploads” to “www-data” so that Apache can upload files to this folder.

~~~~
$cd /var/www/html/web-demo
$ sudo chown -R www-data:www-data uploads
~~~~

In your browser, browse to http://ip-address/web-demo/index.php. You should see that our application is now working. You can login with your name, then upload some photos for testing. (You might have noticed that this demo application does not ask you for a password. This is because we would like to make things as simple as possible. Handling user password is a very complicate issue, which is beyond the scope of this entry level tutorial.) Then I suggest that you spend 10 minutes reading through the demo code index.php. The demo code has reasonable documentation in the form of comments, so I am not going to explain the code here.

**(3) LEVEL 1**

![LEVEL 1](http://www.qyjohn.net/wp-content/uploads/2017/03/Slide4.png)

In this level, we will expand the basic version we have in LEVEL 0 and deploy it to multiple servers. We will do this using two different approaches, the hard approach uses a self-managed NFS server, while the easy approach uses the managed EFS service. 

**STEP 1 - Preparing the EFS File System**

Go to the EFS Console and create an EFS file system. Launch an EC2 instance with Ubuntu 16.04 operating system, then install the following software and mount the EFS file system:

~~~~
$ sudo apt-get update
$ sudo apt-get install nfs-common
$ sudo mkdir /efs 
$ sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 <dns-endpoint-of-your-efs-file-system>:/ /efs
$ sudo chown -R ubuntu:ubuntu /efs
~~~~

Then we add the mounting stuff into /etc/fstab to add the following line, so that you do not need to manually mount the EFS file system when the operating system is rebooted.

~~~~
<dns-endpoint-of-your-efs-file-system>:/  /efs    nfs     rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2    0       0
~~~~

You can verify the above-mentioned configuration is working using the following commands (run them several times):

~~~~
$ df -h
$ mount
$ sudo umount /efs
$ sudo mount /efs
~~~~

**STEP 2 - Install the Apache and PHP**

Run the following commands to install Apache and PHP.

~~~~
$ sudo apt-get update
$ sudo apt-get install apache2 php mysql-client libapache2-mod-php php-mcrypt php-mysql php-curl php-xml php-memcached
$ sudo service apache2 restart
~~~~

Then we use the EFS file system to store our web application.

~~~~
$ cd /var
$ sudo chown -R ubuntu:ubuntu www
$ mv www /efs
$ sudo ln -s /efs/www www
$ cd /var/www/html
$ git clone https://github.com/qyjohn/web-demo
~~~~

**STEP 3 - Launch an RDS Instance**

Launch an RDS instance running MySQL. When launching the RDS instance, create a default database named “web_demo”. When the RDS instance becomes available, connect to the RDS DB server as Root and create a user as you did before for the local DB. This time, when creating the user and granting privileges, you should not use 'username'@'localhost', because the DB will be accessed from web servers but not localhost. You can use 'username'@'%' to create a user that is trusted from any source. Then, use the following command to import the demo data in web_demo.sql to the web_demo database on the RDS database:

~~~~
$ cd /var/www/html/web-demo
$ mysql -h [endpoint-of-rds-instance] -u username -p web_demo < web_demo.sql
~~~~

Now, modify config.php with the new database server hostname, username, password, and database name.

**STEP 4 - Create an ElastiCache Memcached Instance**

We user ElastiCache to resolve the issue about session sharing between multiple web servers. In the ElastiCache console, launch an ElasticCache with Memcache and obtain the endpoint information. On both web servers, install php-memcached and configure php.ini to use memcached for session sharing.

Edit /etc/php/7.0/apache2/php.ini. Make the following modifications:

~~~~
session.save_handler = memcached
session.save_path = "[endpoint-to-the-elasticache-instance]:11211"
~~~~

Then you need to restart Apache on both web servers to make the new configuration effective.

~~~~
$ sudo service apache2 restart
~~~~

**STEP 5 - Create an AMI**

Now, create an AMI from the EC2 instance and launch a new EC2 instance with the AMI.

**STEP 6 - Create an ELB**

Create an ELB and add the two web servers to the ELB. Since we do have Apache running on both web servers, you might want to use HTTP as the ping protocol with 80 as the ping port and “/” as the ping path for the health check parameter for your ELB.

**STEP 7 - Testing*

In your browser, browser to http://elb-endpoint/web-demo/index.php. As you can see, our demo seems to be working on multiple servers. This is so easy!

**(3) LEVEL 2**

![LEVEL 2](http://www.qyjohn.net/wp-content/uploads/2017/03/Slide5.png)

Using a shared file system is probably OK for web applications with reasonably limited traffic, but will be be problematic when the traffic to your site increases. At that point you can scale out your front end to have as many web servers as you want, but your web application is limited by the capability of the shared file system running on a single server. In this level, we will resolve this issue by moving the shared storage from one web server to S3. This way, the web servers only handle your critical business logic, while the images are being served by S3.

We first terminate the web server running NFS client. Then we edit /etc/exports on the web server running NFS server to disable (comment out) the NFS exports and stop nfs-kernel-server service. We no longer need a share file system in the level.

~~~~
$ sudo service nfs-kernel-server stop
~~~~

Next, you will need to edit config.php and make some minor changes. In this configuration file, $s3_bucket is the name of the S3 bucket for share storage, and $s3_baseurl is the URL pointing to the S3 endpoint in the region hosting your S3 bucket. You can find the S3 endpoint from http://docs.aws.amazon.com/general/latest/gr/rande.html. you can also identify this end point in the S3 Console by viewing the properties of an S3 object in the S3 bucket.

~~~~
$storage_option = "s3";
$s3_bucket  = "my_uploads_bucket";
$s3_baseurl = "https://s3-ap-southeast-2.amazonaws.com/";
~~~~

Next we create an AMI from the running instance. Then we create an EC2 Role in the IAM Console, which (IAM Console -> Role -> Create New Role -> Amazon EC2 -> Amazon S3 Full Access). Then we launch a new web server using the AMI and the IAM Role we just created. After the instance becomes “running” and passes health checks, remove the original web server from the ELB (you can terminate it now) and add this new web server to the ELB. In your browser, again browse to http://<dns-name-of-elb>/web-demo/index.php. Currently the only web server behind the ELB is this newly launched instance, but it still has your login information. Newly uploaded images will go to S3 instead of local disk. As the workload of your application increases, you can launch more EC2 instance using the AMI and IAM Role we created just now, and add these new instance to the ELB. When the workload of your application decreases, you can remove some instance from the ELB and terminate them to save cost.

The reason we use IAM Role in this tutorial is that with IAM Role you do not need to supply your AWS Access Key and Secret Key in your code. Rather, Your code will assume the role assigned to the EC2 instance, and access the AWS resources that your EC2 instance is allowed to access. Today many people and organizations host their source code on github.com or some other public repositories. By using IAM roles you no longer hard code your AWS credentials in your application, thus eliminating the possibility of leaking your AWS credentials to the public.

**(4) LEVEL 3**

![LEVEL 3](http://www.qyjohn.net/wp-content/uploads/2017/03/Slide6.png)

Now you have a scalable web application with two web servers, and you know that you can scale in and out the fleet when needed. Is it possible to scale your fleet automatically, according to the workload of your application?

In our demo code, we have a setting to simulate workloads. If you look at config.php, you will find this piece of code:

~~~~
$latency = 0;
~~~~

And, in index.php, there is a corresponding statement:

~~~~
sleep($latency);
~~~~

That is, when a user request index.php, PHP will sleep for $latency seconds. As we know, as the workload increases, the CPU utilization (as well as other factors) increases, resulting in an increase in latency. By manually manipulating the $latency setting, we can simulate heavy workload to your web application, which is reflected in the average latency. It should be noted that you can change the latency simulation settings on each web server. When you have two web servers and you set $latency = 0 on one server and $latency = 1 on the other server, the average latency will be 0.5 second.

With AWS, you can use AutoScaling to scale your server fleet in a dynamic fashion, according to the wordload of the fleet. In this tutorial, we use average latency as a trigger for scaling actions. You can achieve this following these steps:

(1) In your EC2 Console, create a launch configuration using the AMI and the IAM Role that we created in LEVEL 2.

(2) Create an AutoScaling group using the launch configuration we created in step (1), make sure that the AutoScaling group receives traffic from your ELB. Also, change the health check type from EC2 to ELB. (This way, when the ELB determines that an instance is unhealthy, the AutoScaling group will terminate it.) You don’t need to specify any scaling policy at this point.

(3) Click on your ELB and create a new CloudWatch Alarm (ELB -> Monitoring -> Create Alarm) when the average latency is greater than 1000 ms for at least 1 minutes.

(4) Click on your AutoScaling group, and create a new scaling policy (AutoScaling -> Scaling Policies), using the CloudWatch Alarm you just created. The auto scaling action can be “add 1 instance and then wait 300 seconds”. This way, if the average latency of your web application exceeds 1 second, AutoScaling will add one more instance to your fleet.

You can do the testing by adjusting the $latency value on your existing web servers. Please note that to achieve 1000 ms latency, the average value of the $latency settings on all your web servers needs to be greater than 1. So, you can keep $latency = 0 on one of your web servers, and change $latency = 3 on the other web server. This way the average latency will be 1500 ms, which will trigger the CloudWatch Alarm, and hence the auto scaling policy.

When you are done with this step, you can play with scaling down by creating another CloudWatch Alarm and a corresponding auto scaling policy. The CloudWatch alarm will be alarmed when the average latency is smaller than 500 ms for at least 1 minute, and the auto scaling action can be “remove 1 instance and then wait 300 seconds”.

**(5) LEVEL 4**

![LEVEL 4](http://www.qyjohn.net/wp-content/uploads/2017/03/Slide7.png)

For many web applications, database can be a serious bottleneck. In our photo sharing demo, usually the number of view image requests is much greater than the number of upload requests. It is very possible that for many view requests, the most recent N images are actually the same. However, we are connecting to the database to fetch records for the most recent N images for each and every view request. It would be reasonable to update the images we show on the index page in an incremental way, for example, every 1 or 2 minutes.

In this level, we will add a cache layer (cache server) between the web servers and the database. When we fetch records for the most recent N images from the database, we cache it in the cache server. When there is a new view request coming in, we no longer connect to the database, but return the cached result to the user. When there is a new image upload, we invalidate the cached version by deleting it from the cache server. When the next view request comes, we fetch records for the most recent N images from the database again, then cache this new version to the cache server. This way the cache version is always up-to-date.

The demo code has support for database caching through ElastiCache, using the same ElastiCache instance for session sharing. This caching behavior is not enable by default. You can edit config.php on all web servers with details regarding the cache server:

~~~~
$enable_cache = true;
$cache_server  = "dns-name-of-your-elasticache-instance";
~~~~

Refresh the demo application in your browser, you will see that the “Fetching N records from the database.” message is now gone, indicating that the information you are seeing is obtained from ElastiCache. When you upload a new image, you will see this message again, indicating the cache is being updated.

The following code is responsible of handling this cache logic:

~~~~
// Get the most recent N images
if ($enable_cache)
{
	// Attemp to get the cached records for the front page
	$mem = open_memcache_connection($cache_server);
	$images = $mem->get("front_page");
	if (!$images)
	{
		// If there is no such cached record, get it from the database
		$images = retrieve_recent_uploads($db, 10);
		// Then put the record into cache
		$mem->set("front_page", $images, time()+86400);
	}
}
else
{
	// This statement get the last 10 records from the database
	$images = retrieve_recent_uploads($db, 10);
}
~~~~

Also pay attention to this code when doing image uploads. We deleted the cache after the user uploads an images. This way, when the next request comes in, we will fetch the latest records from the database, and put them into the cache again.

~~~~
	if ($enable_cache)
	{
		// Delete the cached record, the user will query the database to 
		// get an updated version
		$mem = open_memcache_connection($cache_server);
		$mem->delete("front_page");
	}
~~~~

**(7) Others**

In this tutorial, we build a scalable web application using various AWS services including EC2, RDS, S3, ELB, CloudWatch, AutoScaling, IAM, and ElastiCache. It demonstrates how easy it is to build a scalable web application that can scale reasonably well using the various AWS building blocks. It should be noted that the code being used in this tutorial is for demo only, and can not be used in a production system.

You are encouraged to explore the following topics:

(1) How do you trouble shoot issues when things does not work in this demo. For example, your application is unable to connect to the RDS instance, or the ElastiCache. What is needed to make things work?

(2) How to identify bottleneck when there is a performance issue? How to enhance the performance of this demo application?

(3) How to make the deployment process easier?

(4) How to make this demo application more secure?

(5) Any other topic that you would like to explore.
