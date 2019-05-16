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

(2) Off load user uploads to S3.

(3) Dynamically scale the size of your server fleet according to the actual traffic to your web application.

(4) Implement a cache layer between the web server and the database server.

(5) Use CloudFront for content delivery.

(6) Use Kinesis Analytics to perform simple near realtime analysis for your web traffic. 

**(2) LEVEL 0**

![LEVEL 0](http://www.qyjohn.net/wp-content/uploads/2017/03/Slide3.png)

In this level, we will build a basic version with all the components deployed on one single server. The web application is developed with PHP, using Apache as the web serer, with MySQL as the database to store user upload information. You do not need to write any code, because I have a set of demo code prepared for you. You just need to launch an EC2 instance, carry out some basic configurations, then deploy the demo code.

Login to your AWS Console and navigate to the EC2 Console. Launch an EC2 instance with an Ubuntu 18.04 AMI. Make sure that you allocate a public IP to your EC2 instance. In your security group settings, open port 80 for HTTP and port 22 for SSH access. After the instance becomes “running” and passes health checks, SSH into your EC2 instance to setup software dependencies and download the demo code from Github to the default web server folder:

~~~~
$ sudo apt-get update
$ sudo apt-get install git mysql-server
$ sudo mysql_secure_installation
~~~~

~~~~
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
$ cd web-demo
~~~~

Change the ownership of folder “uploads” to “www-data” so that Apache can upload files to this folder.

~~~~
$ sudo chown -R www-data:www-data uploads
~~~~

Then we create a MySQL database and a MySQL user for our demo. Here we use “web_demo” as the database name, and “username” as the MySQL user.

~~~~
$ sudo mysql
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

Use a text editor to open config.php, then change the username and password for your MySQL installation. The example commands before created a DB user named "username" with the password "password". This is the default DB credentials configured in config.php. If you used a different username or password, update config.php accordingly.

In your browser, browse to http://ip-address/web-demo/index.php. You should see that our application is now working. You can login with your name, then upload some photos for testing. (You might have noticed that this demo application does not ask you for a password. This is because we would like to make things as simple as possible. Handling user password is a very complicate issue, which is beyond the scope of this entry level tutorial.) Then I suggest that you spend 10 minutes reading through the demo code index.php. The demo code has reasonable documentation in the form of comments, so I am not going to explain the code here.

Now you are able to get your website working, please upload some more pictures for testing. Upload some small pictures and some big pictures (like 20 MB) to see what happens. Fix any issues you may observe in the tests.

**(3) LEVEL 1**

![LEVEL 1](http://www.qyjohn.net/wp-content/uploads/2017/02/Level_1.png)

In this level, we will expand the basic version we have in LEVEL 0 and deploy it to multiple servers. We will do this using two different approaches, the hard approach uses a self-managed NFS server, while the easy approach uses the managed EFS service. 

**STEP 1 - Preparing the EFS File System**

Go to the EFS Console and create an EFS file system. 

Terminate the previous EC2 instance because we no longer need it. Launch a new EC2 instance with the Ubuntu 18.04 operating system. SSH into the EC2 instance to install the following software and mount the EFS file system:

~~~~
$ sudo apt-get update
$ sudo apt-get install nfs-common
$ sudo mkdir /efs 
$ sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 <dns-endpoint-of-your-efs-file-system>:/ /efs
$ sudo chown -R ubuntu:ubuntu /efs
~~~~

Then we add the mounting stuff into /etc/fstab to add the following line, so that you do not need to manually mount the EFS file system when the operating system is rebooted.

~~~~
<dns-endpoint-of-your-efs-file-system>:/  /efs    nfs    auto,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2    0       0
~~~~

You can verify the above-mentioned configuration is working using the following commands (run them several times):

~~~~
$ df -h
$ mount
$ sudo umount /efs
$ sudo mount /efs
~~~~

**STEP 2 - Install the Apache and PHP**

Run the following commands to install Apache and PHP. Notice that we are not installing the MySQL server this time. 

~~~~
$ sudo apt-get install apache2 php mysql-client libapache2-mod-php php-mcrypt php-mysql php-curl php-xml
$ sudo service apache2 restart
~~~~

Then we use the EFS file system to store our web application.

~~~~
$ cd /efs
$ git clone https://github.com/qyjohn/web-demo
$ cd web-demo
$ sudo chown -R www-data:www-data uploads
$ cd /var/www/html
$ sudo ln -s /efs/web-demo web-demo
~~~~

**STEP 3 - Launch an RDS Instance**

Launch an RDS instance running MySQL. When launching the RDS instance, create a default database named “web_demo”. When the RDS instance becomes available. Please make sure that the security group being used on the RDS instance allows inbound connection from your EC2 instance. Then, connect to the RDS instance and create a user for your application. This time, when granting privileges, you need to grant external access for the user.

~~~~
$ mysql -h [endpoint-of-rds-instance] -u root -p
mysql> CREATE DATABASE web_demo;
mysql> CREATE USER 'username'@'%' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON web_demo.* TO 'username'@'%';
mysql> quit
~~~~

Then, use the following command to import the demo data in web_demo.sql to the web_demo database on the RDS database:

~~~~
$ cd /var/www/html/web-demo
$ mysql -h [endpoint-of-rds-instance] -u username -p web_demo < web_demo.sql
~~~~

Now, modify config.php with the new database server hostname, username, password, and database name.

**STEP 4 - Create an ElastiCache Memcached Cluster**

We use ElastiCache to resolve the session sharing issue between multiple web servers. In the ElastiCache console, launch an ElastiCache cluster with Memcached (just 1 single node is enough) and obtain the endpoint information. Please make sure that the security group being used on the ElastiCache cluster allows inbound connection from your EC2 instance. 

On the web server, configure php.ini to use Memcached for session sharing.

Edit /etc/php/7.0/apache2/php.ini. Make the following modifications:

~~~~
session.save_handler = memcached
session.save_path = "[dns-endpoint-to-the-elasticache-node]:11211"
~~~~

Then you need to restart Apache the web server to make the new configuration effective.

~~~~
$ sudo apt-get install php-memcached
$ sudo service apache2 restart
~~~~

If you create an ElastiCache Memcached cluster with multiple nodes, your configuration would look like this:

~~~~
session.save_handler = memcache
session.save_path = "tcp://elasticache-node1:11211,tcp://elasticache-node2:11211,tcp://elasticache-node3:11211"
~~~~

Edit /etc/php/7.0/mods-available/memcached.ini, add the following two lines to support session redundancy. Please note that the value of memcache.session_redundancy equals to the number of cache nodes plus 1 (because of a [bug](https://bugs.php.net/bug.php?id=58585) in PHP. 

~~~~
memcache.allow_failover=1
memcache.session_redundancy=4
~~~~

Then you need to restart Apache the web server to make the new configuration effective.

~~~~
$ sudo apt-get install php-memcache
$ sudo service apache2 restart
~~~~

**STEP 4b - (Optional) Create an ElastiCache Redis Cluster**

If you don't want to use MemCached, you can Redis to resolve the session sharing issue between multiple web servers. In the ElastiCache console, launch an ElastiCache cluster with Redis with the "Cluster Mode enabled" option and obtain the configuration endpoint information. Please make sure that the security group being used on the ElastiCache cluster allows inbound connection from your EC2 instance. 

On the web server, configure php.ini to use Redis for session sharing.

Edit /etc/php/7.0/apache2/php.ini. Make the following modifications:

~~~~
session.save_handler = rediscluster
session.save_path = "seed[]=[configuration-endpoint-of-the-elasticache-redis-cluster:6379]"
~~~~

If you have not yet installed the php-redis module, you need to install it to make things work. Then you need to restart Apache the web server to make the new configuration effective.

~~~~
$ sudo apt-get install php-redis
$ sudo service apache2 restart
~~~~

**STEP 5 - Create an AMI**

Now, create an AMI from the EC2 instance and launch a new EC2 instance with the AMI.

**STEP 6 - Create an ELB**

Create an Application Load Balancer (ALB) and register the two EC2 instances to the ALB target group. Since we do have Apache running on both web servers, you might want to use HTTP as the ping protocol with 80 as the ping port and “/” as the ping path for the health check parameter for your ELB.

**STEP 7 - Testing**

In your browser, browser to http://elb-endpoint/web-demo/index.php. As you can see, our demo seems to be working on multiple servers. This is so easy!

**(3) LEVEL 2**

![LEVEL 2](http://www.qyjohn.net/wp-content/uploads/2017/03/Slide5.png)

Using a shared file system is probably OK for web applications with reasonably limited traffic, but will be be problematic when the traffic to your site increases. At that point you can scale out your front end to have as many web servers as you want, but your web application is limited by the capability of the shared file system. In this level, we will resolve this issue by moving the storage from EFS to S3. This way, the web servers only handle your critical business logic, while the images are being served by S3.

From one of the EC2 instance, edit config.php and make some minor changes. It does not matter from which EC2 instance you make the changes, because the source files are stored on EFS. The changes will be reflected on both EC2 instances. 

In the following configuration, $s3_bucket is the name of the S3 bucket for share storage, and $s3_baseurl is the URL pointing to the S3 endpoint in the region hosting your S3 bucket. You can find the S3 endpoint from http://docs.aws.amazon.com/general/latest/gr/rande.html. you can also identify this end point in the S3 Console by viewing the properties of an S3 object in the S3 bucket.

It is very important that your S3 bucket does not "Block new public ACLs and uploading public objects". You can view the configurations of your S3 bucket in the S3 console under "Permissions -> Public Access Settings".

~~~~
$s3_region  = "us-east-2";
$s3_bucket  = "your_s3_bucket_name";
$s3_prefix  = "uploads";
$s3_baseurl = "https://s3-us-east-2.amazonaws.com/";
~~~~

In order for this new setting to work, we need to attach an IAM Role to both EC2 instances. In the IAM Console, create an EC2 Role in the IAM Console, which allows full access to S3 (IAM Console -> Role -> Create Role -> AWS Service -> EC2 -> EC2 -> Amazon S3 Full Access). In the EC2 Console, attach the newly created role to both EC2 instances one by one (Instance -> Actions -> Instance Settings -> Attach/Replace IAM Role).

In your browser, again browse to http://<dns-name-of-elb>/web-demo/index.php. At this point you will see missing images, because the previously uploaded images are not available on S3. Newly uploaded images will go to S3 instead of local disk. 

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

That is, when a user request index.php, PHP will sleep for $latency seconds. As we know, as the workload increases, the CPU utilization (as well as other factors) increases, resulting in an increase in latency (response time). By manually manipulating the $latency setting, we can simulate heavy workload to your web application, which is reflected in the average latency. 

With AWS, you can use AutoScaling to scale your server fleet in a dynamic fashion, according to the workload of the fleet. In this tutorial, we use average latency (response time) as a trigger for scaling actions. You can achieve this following these steps:

(1) In your EC2 Console, create a launch configuration using the AMI and the IAM Role that we created in LEVEL 2.

(2) Create an AutoScaling group using the launch configuration we created in step (1), make sure that the AutoScaling group receives traffic from your ALB target group. Also, change the health check type from EC2 to ELB. (This way, when the ELB determines that an instance is unhealthy, the AutoScaling group will terminate it.) You don’t need to specify any scaling policy at this point.

(3) Click on your ELB and create a new CloudWatch Alarm (ELB -> Monitoring -> Create Alarm) when the average latency (response time) is greater than 1000 ms for at least 1 minutes.

(4) Click on your AutoScaling group, and create a new scaling policy (AutoScaling -> Scaling Policies), using the CloudWatch Alarm you just created. The auto scaling action can be “add 1 instance and then wait 300 seconds”. This way, if the average latency of your web application exceeds 1 second, AutoScaling will add one more instance to your fleet.

You can do the testing by adjusting the $latency value on your existing web servers. Please note the source code resides on your EFS file system, when you make the change from one of your EC2 instances, the change is reflected on all of your EC2 instances.

When you are done with this step, you can play with scaling down by creating another CloudWatch Alarm and a corresponding auto scaling policy. The CloudWatch alarm will be alarmed when the average latency is smaller than 500 ms for at least 1 minute, and the auto scaling action can be “remove 1 instance and then wait 300 seconds”.

**(5) LEVEL 4**

![LEVEL 4](http://www.qyjohn.net/wp-content/uploads/2017/03/Slide7.png)

For many web applications, database can be a serious bottleneck. In our photo sharing demo, usually the number of view image requests is much greater than the number of upload requests. It is very possible that for many view requests, the most recent N images are actually the same. However, we are connecting to the database to fetch records for the most recent N images for each and every view request. It would be reasonable to update the images we show on the index page in an incremental way, for example, every 1 or 2 minutes.

In this level, we will add a cache layer (cache server) between the web servers and the database. When we fetch records for the most recent N images from the database, we cache it in the cache server. When there is a new view request coming in, we no longer connect to the database, but return the cached result to the user. When there is a new image upload, we invalidate the cached version by deleting it from the cache server. When the next view request comes, we fetch records for the most recent N images from the database again, then cache this new version to the cache server. This way the cache version is always up-to-date.

The demo code has support for database caching through ElastiCache, using the same ElastiCache instance for session sharing. This caching behavior is not enable by default. You can edit config.php on all web servers with details regarding the cache server:

~~~~
// Cache configuration
$enable_cache = true;
$cache_type = "memcached";	// memcached or redis
$cache_key  = "images_html";
if ($enable_cache && ($cache_type == "memcached"))
{
	$cache = open_memcache_connection();
}
else if ($enable_cache && ($cache_type == "redis"))
{
	$cache = open_redis_connection();
}
~~~~

Refresh the demo application in your browser, you will see that the “Fetching N records from the database.” message is now gone, indicating that the information you are seeing is obtained from ElastiCache. When you upload a new image, you will see this message again, indicating the cache is being updated.

The following code is responsible of handling this cache logic:

~~~~
// Get the most recent N images
if ($enable_cache)
{
	// Attemp to get the cached records for the front page
	$images_html = $cache->get($cache_key);
	if (!$images_html)
	{
		// If there is no such cached record, get it from the database
		$images = retrieve_recent_uploads($db, 10, $storage_option);
		// Convert the records into HTML
		$images_html = db_rows_2_html($images, $storage_option, $hd_folder, $s3_bucket, $s3_baseurl);
		// Then put the HTML into cache
		$cache->set($cache_key, $images_html);
	}
}
else
{
	// This statement get the last 10 records from the database
	$images = retrieve_recent_uploads($db, 10, $storage_option);
	$images_html = db_rows_2_html($images, $storage_option, $hd_folder, $s3_bucket, $s3_baseurl);
}
// Display the images
echo $images_html;
~~~~

Also pay attention to this code when doing image uploads. We deleted the cache after the user uploads an images. This way, when the next request comes in, we will fetch the latest records from the database, and put them into the cache again.

~~~~
		if ($enable_cache)
		{
			// Delete the cached record, the user will query the database to get an updated version
			if ($cache_type == "memcached")
			{
				$cache->delete($cache_key);
			}
			else if ($cache_type == "redis")
			{
				$cache->del($cache_key);
			}
		}
~~~~

**(6) LEVEL 5**

![Web Demo](http://www.qyjohn.net/wp-content/uploads/2015/01/Level_5.png)

In this level, we create a CloudFront distribution with your S3 bucket as the origin. This way your static content is served to your end users from the nearest edge locations. In your code, you only need to make the following tiny changes.

~~~~
$enable_cf  = true;
$cf_baseurl = "http://xxxxxxxxxxxxxx.cloudfront.net/";
~~~~

Reload the web page in your browser to observe the behavior. Are you able to use CloudFront when the uploaded pictures are stored on disk?

**(7) LEVEL 6**

![Web Demo](http://www.qyjohn.net/wp-content/uploads/2015/01/Level_6.png)

In this level, we will look into how we can perform real-time log analysis for your web application. This is achieve using the Kinesis data stream and Kinesis Analytics application. 

First of all, we need to create two Kinesis data streams (using the Kinesis web console) in the us-east-1 region: web-access-log (1 shard is sufficient for our demo).

SSH into your EC2 instance, configure your Apache to log in JSON format. This will make it easier for Kinesis Analytics to work with your logs. Edit /etc/apache2/apache2.conf, find the area with LogFormat, and add the following new log format to it. For more information on custom log format for Apache, please refer to [Apache Module mod_log_config](http://httpd.apache.org/docs/current/mod/mod_log_config.html).

~~~~
LogFormat "{ \"request_time\":\"%t\", \"client_ip\":\"%a\", \"client_hostname\":\"%V\", \"server_ip\":\"%A\", \"request\":\"%U\", \"http_method\":\"%m\", \"status\":\"%>s\", \"size\":\"%B\", \"userAgent\":\"%{User-agent}i\", \"referer\":\"%{Referer}i\" }" kinesis
~~~~

Then edit /etc/apache2/sites-available/000-default.conf, change the CustomLog line to use your own log format:

~~~~
CustomLog ${APACHE_LOG_DIR}/access.log kinesis
~~~~

Restart Apache to allow the new configuration to take effect:

~~~~
$ sudo service apache2 restart
~~~~

Then, install and configure the Kinesis Agent:

~~~~
$ cd ~
$ sudo apt-get install openjdk-8-jdk
$ git clone https://github.com/awslabs/amazon-kinesis-agent
$ cd amazon-kinesis-agent
$ sudo ./setup --install
~~~~

After the agent is installed, the configuration file can be found in /etc/aws-kinesis/agent.json. Edit the configuration file to send your Apache access log to the web-access-log stream. (Let's not worry about the error log in this tutorial.)

~~~~
{
  "cloudwatch.emitMetrics": true,
  "kinesis.endpoint": "kinesis.us-east-1.amazonaws.com",
  "firehose.endpoint": "",
  
  "flows": [
    {
      "filePattern": "/var/log/apache2/access.log",
      "kinesisStream": "web-access-log",
      "partitionKeyOption": "RANDOM"
    }
  ]
}
~~~~

Once you updated the configuration file, you can start the Kinesis Agent using the following command:

~~~~
$ sudo service aws-kinesis-agent stop
$ sudo service aws-kinesis-agent start
~~~~

Then you can check the status of the Kinesis Agent using the following command:

~~~~
$ sudo service aws-kinesis-agent status
~~~~

If the agent is not working as expected, look into the logs (under /var/log/aws-kinesis-agent) to understand what is going on. (If there is no log, what would you do?) It is likely that the user running the Kinesis Agent (aws-kinesis-agent-user) does not have access to the Apache logs (/var/log/apache2/). To resolve this issue, you can add the aws-kinesis-agent-user to the adm group.

~~~~
$ sudo usermod -a -G adm aws-kinesis-agent-user 
$ sudo service aws-kinesis-agent stop
$ sudo service aws-kinesis-agent start
~~~~

Refresh your web application in the browser, then watch the Kinesis Agent logs to see whether your logs are pushed to the Kinesis streams. When the Kinesis Agent says the logs are successfully sent to destinations, check the "Monitoring" tab in the Kinesis data streams console to confirm this.

Create a new AMI from the above-mentioned EC2 instance, then create a new launch configuration from the new AMI. Modify your Auto Scaling group to use the new launch configuration. This way, all of the EC2 instance in your web server fleet is capable of sending logs to your Kinesis stream.

If you are tired of manually refreshing your web browser, you can use the Apache Benchmark tool (ab) to generate the web traffic automatically.

~~~~
$ ab -n 100000 -c 2 http://<dns-endpoint-of-your-load-balancer>/web-demo/index.php
~~~~

Now go to the Kinesis Analytics console to create a Kinesis Analytics Application, with the web-access-log data stream as the source. Click on the "Discover scheme" to automatically discover the scheme in the data, then save the scheme and continue. In the SQL Editor, copy and paste the following sample SQL statements. Then click on the "Save and run SQL" button to start your application.

~~~~
-- Create a destination stream
CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM" (client_ip VARCHAR(16), request_count INTEGER);
-- Create a pump which continuously selects from a source stream (SOURCE_SQL_STREAM_001)
CREATE OR REPLACE PUMP "STREAM_PUMP" AS INSERT INTO "DESTINATION_SQL_STREAM"
-- Aggregation functions COUNT|AVG|MAX|MIN|SUM|STDDEV_POP|STDDEV_SAMP|VAR_POP|VAR_SAMP
SELECT STREAM "client_ip", COUNT(*) AS request_count
FROM "SOURCE_SQL_STREAM_001"
-- Uses a 10-second tumbling time window
GROUP BY "client_ip", FLOOR(("SOURCE_SQL_STREAM_001".ROWTIME - TIMESTAMP '1970-01-01 00:00:00') SECOND / 10 TO SECOND);
~~~~

From multiple EC2 instances, use the Apache Benchmark tool (ab) to generate some more web traffic. Observe and explain the query results in the Kinesis Analytics console.

~~~~
$ ab -n 100000 -c 2 http://<dns-endpoint-of-your-load-balancer>/web-demo/index.php
~~~~

**(8) Others**

In this tutorial, we build a scalable web application using various AWS services including EC2, RDS, S3, ELB, CloudWatch, AutoScaling, IAM, and ElastiCache. It demonstrates how easy it is to build a scalable web application that can scale reasonably well using the various AWS building blocks. It should be noted that the code being used in this tutorial is for demo only, and can not be used in a production system.

You are encouraged to explore the following topics:

(1) How do you trouble shoot issues when things does not work in this demo. For example, your application is unable to connect to the RDS instance, or the ElastiCache. What is needed to make things work?

(2) How to identify bottleneck when there is a performance issue? How to enhance the performance of this demo application?

(3) How to make the deployment process easier?

(4) How to make this demo application more secure?

(5) Use Kinesis Firehose to delivery your logs to S3, Elastic Search, or Splunk for further analysis, search, etc.
