# ec2-autoscaling-test - LAMP web server

Demonstrate AWS auto-scaling using a LAMP web server with a simple php script

## Todo

### Instance Information

* Region: us-east-1
* AZ: us-east-1d
* AMI: ami-abd8d5c2
    * based on Amazon Linux AMI 2013.09.2 - ami-bba18dd2
* Size: t1.micro
* Root storage: EBS
* Security group
    * Name: sshandhttp
    * ID: sg-4dedf426
    * Rules: ports 22 and 80 are open to the world (well, right now everything is open to the world while I test :p)

#### Elastic Load Balancer

* Scheme: internet-facing
* Port Configuration: 80 (HTTP) forwarding to 80 (HTTP)
* Ping Target: HTTP:80:/phpinfo.php
* Unhealthy Threshold: 2
* Timeout: 5
* Healthy Threshold: 10
* Interval: 0.5

### AMI setup

#### Install LAMP stack

Note: This setup is taken from one of Amazon's examples: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html#d0e33722.

From the base Amazon Linux AMI, run the following as root (or add to the instance user data, or put in a script):

	#!/bin/bash
	yum update -y
	yum groupinstall -y "Web Server" "MySQL Database" "PHP Support"
	yum install -y php-mysql
	service httpd start
	chkconfig httpd on
	groupadd www
	usermod -a -G www ec2-user
	chown -R root:www /var/www
	chmod 2775 /var/www
	find /var/www -type d -exec chmod 2775 {} +
	find /var/www -type f -exec chmod 0664 {} +
	echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php

Since I have port 80 open, http://my.public.dns.amazonaws.com/phpinfo.php returns the PHP information.

#### Add a CPU-intensive PHP script

It just needs to do anything long enough to stress the CPU. How about counting to 1,000,000,000?

	<?php $i = 0; while ($i < 1000000000){ $i++; } ?>

Or run the following as root to create the file.

	echo "<?php \$i = 0; while (\$i < 1000000000){ \$i++; } ?>" > /var/www/html/stress.php

A quick test in my browser shows that http://my.public.dns.amazonaws.com/stress.php will bring CPU User above 99% for about 15 seconds.

If we tell our Auto Scaling group to add a new instance whenever the average of CPU utilization is greater than 10% over 1 minute, a single call to stress.php should trigger an additional instance.

### Auto Scaling

Launch configuration is as described above in Instance Information. The Elastic Load Balancer should be provisioned before creating the Auto Scaling group.

Scaling group is defined as:

* min: 1 instance
* max: 2 instances
* execute policy: when average CPU Utilization >= 10% over 60 seconds
    * increase group size by 1 instance
    * wait 60 seconds before allowing another scaling activity
* execute policy: when average CPU Utilization < 10% over 60 seconds
    * decrease group size by 1 instance
    * wait 60 seconds before allowing another scaling activity

#### Test Auto Scaling

