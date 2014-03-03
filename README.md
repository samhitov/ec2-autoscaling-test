# ec2-autoscaling-test

Demonstrate AWS auto-scaling using a simple node.js web server

## Todo

* Start node.js on boot, so autoscaling is useful
* Put webs behind an ELB
* Make the node.js code do something taxing enough so external requests can cause load

### Instance Information

* Region: us-east-1
* AZ: us-east-1d
* AMI: ami-c1575aa8
    * based on Amazon Linux AMI 2013.09.2 - ami-bba18dd2 (just with a simple node.js server installed)
* Size: t1.micro
* Root storage: EBS
* Security group
    * Name: sshandhttp
    * ID: sg-4dedf426
    * Rules: ports 22 and 80 are open to the world (well, right now everything is open to the world while I test :p)

### AMI setup

#### Install

Note: haven't used node.js before, so I'm taking liberally from http://iconof.com/blog/how-to-install-setup-node-js-on-amazon-aws-ec2-complete-guide

From the base Amazon Linux AMI, run:

	sudo yum -y update
	# in production, would pick and choose packages, but this is easy for testing
	sudo yum -y groupinstall 'Development Tools'
	git clone git://github.com/joyent/node.git
	cd node
	./configure
	make
	sudo make install
	# using visudo, add :/usr/local/bin to the end of "Defaults    secure_path"
	cd ~
	# can use this to hackishly start a running server
	sudo npm install forever -g

#### Test

Note: Anything using port 80 requires sudo. Standard practice seems to avoid this by running node on a nonstandard port like 8080, then forwarding 80 to 8080.

Test with a simple example adapted from http://nodejs.org:

    var http = require('http');
    http.createServer(function (req, res) {
      res.writeHead(200, {'Content-Type': 'text/plain'});
      res.end('Hello World\\n');
    }).listen(80);
    console.log('Server is running');

Save as example.js, then run:

    sudo node example.js
    Server is running

#### Keep node.js running using forever

iptables was giving me trouble, not sure why. I'm just shutting it off for now, while I get the test working.

    sudo service iptables stop
    sudo forever start -al forever.log -ao out.log -ae err.log example.js

### Auto Scaling

Launch configuration is as described above in Instance Information.

Security group is defined as:

* min: 1 instance
* max: 2 instances
* execute policy: when average CPU Utilization >= 10% over 300 seconds
    * increase group size by 1 instance
    * wait 60 seconds before allowing another scaling activity
* execute policy: when average CPU Utilization < 10% over 300 seconds
    * decrease group size by 1 instance
    * wait 60 seconds before allowing another scaling activity

#### Test Auto Scaling

This is a t1.micro, so I can tax it with just about anything in a while loop, i.e.:

    while true; do sudo du /; done

Confirmed with scaling history that a new instance was added. Killed the 'du's, and an instance was terminated 5 minutes later.
