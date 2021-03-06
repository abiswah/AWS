## Creating/Launching Application using Terraform.
### Task Description:-
1. Create the key and security group which allow the port 80.
2. Launch EC2 instance.
3. In this Ec2 instance use the key and security group which we have created in step 1.
4. Launch one Volume (EBS) and mount that volume into /var/www/html
5. Developer have uploded the code into github repo also the repo has some images.
6. Copy the github repo code into /var/www/html
7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.
8. Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to  update in code in /var/www/html

### Following the given steps anyone can achieve the desired results:-
#### Step 1.
In the first step you have to declare your provider and its necessary login credentials, example:-
```
provider "aws" {
  region     = "ap-south-1"
  profile    = "abhi"
}
```
#### Step 2.
In the second step you have to create a security group which allows port number 80, which in turns provide the services for HTTP protocol and port number 22 which provides services for SSH protocol. Egress is not open for all IP's and all ports. Also, CIDR is configured for IPv4 not for IPv6. The following commands will perform the above query:-
```
resource "aws_security_group" "allow_http" {
  name        = "allow_http"
  description = "Allow HTTP SSH inbound traffic"
  vpc_id      = "vpc-d4ebf6bc"

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [ "0.0.0.0/0" ]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags = {
    Name = "my_http"
  }
}
```
#### Step 3.
In the third and the most critical step as all the steps above revolves around this step. The instance which we are creating here is used to deploy webserver and nearly all other tasks are also done here. The isntance is launched using the keys and security groups created previously.
```
resource "aws_instance" "myweb" {
	ami		= "ami-005956c5f0f757d37"
	instance_type	="t2.micro"
	key_name          = "abhishek"
  	security_groups   = [ "allow_http" ]

	 connection {
    	type        = "ssh"
    	user        = "ec2-user"
    	private_key = file("C:/Users/Abhishek/Downloads/abhishek.pem")
    	host        = "${aws_instance.myweb.public_ip}"
  	}
  
 	 provisioner "remote-exec" {
    	inline = [
      	"sudo yum install httpd  -y",
      	"sudo service httpd start",
      	"sudo service httpd enable"
    	]
 	 }

	tags = {
		Name = "Abhishekos"
	}
}

output "o3" {
	value = aws_instance.myweb.public_ip
}

output "o4" {
	value = aws_instance.myweb.availability_zone
}
```
#### Step 4.
Next, i created an S3 bucket and also manipulated the terraform to save my bucket name in my local system inside my working repository. This is for the time when I destroy the infrastructure created by, it will ask for bucket name which I have made dynamic as we need a unique name for our bucket as S3 is a global service.
```
resource "aws_s3_bucket" "myuniquebucket1227" {
  bucket = "myuniquebucket1227" 
  acl    = "public-read"
  tags = {
    Name        = "uniquebucket1227" 
  }
  versioning {
	enabled =true
  }
}

resource "aws_s3_bucket_object" "s3object" {
  bucket = "${aws_s3_bucket.myuniquebucket1227.id}"
  key    = "1076883.jpg"
  source = "C:/Users/Abhishek/Downloads/1076883.jpg"
}
```
#### Step 5.
Creating Cloudfront distribution for S3 bucket in this step as we want to decrease the latency as much as we can through the CDN (Content Delivery Network). This in turns provide a different link to all S3 storage contents and will also help in reducing latency for the clients.
```
resource "aws_cloudfront_distribution" "imagecf" {
    origin {
        domain_name = "myuniquebucket1227.s3.amazonaws.com"
        origin_id = "S3-myuniquebucket1227"


        s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.origin_access_identity.cloudfront_access_identity_path
    }
  }
       
    enabled = true
      is_ipv6_enabled     = true

    default_cache_behavior {
        allowed_methods = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
        cached_methods = ["GET", "HEAD"]
        target_origin_id = "S3-myuniquebucket1227"


        # Forward all query strings, cookies and headers
        forwarded_values {
            query_string = false
        
            cookies {
               forward = "none"
            }
        }
        viewer_protocol_policy = "allow-all"
        min_ttl = 0
        default_ttl = 10
        max_ttl = 30
    }
    # Restricts who is able to access this content
    restrictions {
        geo_restriction {
            # type of restriction, blacklist, whitelist or none
            restriction_type = "none"
        }
    }


    # SSL certificate for the service.
    viewer_certificate {
        cloudfront_default_certificate = true
    }
}
```
#### Step 6.
In this step we are creating an extra EBS (Elastic Block Storage) volume and attaching this extra created volume to our instances so that it can be accessed by us from instance. 
```
resource "aws_ebs_volume" "ebs_vol" {
  availability_zone = aws_instance.myweb.availability_zone
  size              = 1

  tags = {
    Name = "mypendrive"
  }
}

resource "aws_volume_attachment" "EBS_attachment" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.ebs_vol.id
  instance_id = aws_instance.myweb.id
  force_detach = true
}

output "myos_ip" {
  value = aws_instance.myweb.public_ip
}

resource "null_resource" "null1"  {

  depends_on = [
    aws_volume_attachment.EBS_attachment,
  ]
  connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key = file("C:/Users/Abhishek/Downloads/abhishek.pem")
    host     = aws_instance.myweb.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo mkfs.ext4  /dev/xvdf",
      "sudo mount  /dev/xvdf  /var/www/html",
      "sudo rm -rf /var/www/html/*",
      "sudo git clone https://github.com/abiswah/AWS/blob/master/Firstproject.html"
    ]
  }
}
```
### Here are few screenshots of my task and its output:-
![alt text](https://github.com/abiswah/AWS/blob/master/image_7808d6e1-e3d6-4eb8-9f21-81f492c3f81220201018_161414.jpg)
![alt text](https://github.com/abiswah/AWS/blob/master/image_52924011-ebfa-48d4-8630-0920e357a7dd20201018_161420.jpg)
![alt text](https://github.com/abiswah/AWS/blob/master/image_2942cad9-7f66-4e60-9a60-5170fa70dd1520201018_161433.jpg)
![alt text](https://github.com/abiswah/AWS/blob/master/image_920f1ed2-f1d0-4022-9255-f8b8bc9f27ea20201018_161435.jpg)
![alt text](https://github.com/abiswah/AWS/blob/master/image_6caf366a-0156-40d1-b22c-8c7ce778317f20201018_161437.jpg)
![alt text](https://github.com/abiswah/AWS/blob/master/image_7fd42a92-dc00-46dd-b09d-774d1f6ef1ba20201018_161440.jpg)
![alt text](https://github.com/abiswah/AWS/blob/master/image_939fd662-9d42-4e56-b286-d74225510df720201018_161439.jpg)
![alt text](https://github.com/abiswah/AWS/blob/master/image_2efe19c9-0435-45f8-9c7e-af907d348aae20201018_161442.jpg)

### Thanks for reading.....
