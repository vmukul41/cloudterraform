AUTOMATED IAC using terraform
1. Create the key and security group which allow the port 80.
//security group creation

resource "aws_security_group" "security_group" {
  name        = "allow_httpd_ssh"
  description = "Allow http and ssh traffic"
  vpc_id      = "vpc-c2ccd0aa"
 

  ingress {
    description = "ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "http"
    protocol   = "tcp"
    from_port  = 80
    to_port    = 80
    cidr_blocks = ["0.0.0.0/0"]
  }
 
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow_http_request"
  }
}
2. Launch EC2 instance.
3. In this Ec2 instance use the key and security group which we have created in step 1.
// instance creation
resource "aws_instance" "webserver" {

depends_on = [
       aws_security_group.security_group
    ]

  ami           = "ami-0447a12f28fddb066"
  instance_type = "t2.micro"
  key_name = "mykey3333"
  security_groups =  [ "allow_httpd_ssh" ]

  tags = {
    Name = "totalautomated"
  }



  connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key  = file("C:/Users/hello/Downloads/mykey3333.pem")
    host     = aws_instance.webserver.public_ip 
  }
 

  provisioner "remote-exec" {
    inline = [ 
               "sudo yum install httpd -y",
               "sudo yum install php -y",
               "sudo yum install git -y" ,
               "sudo systemctl enable httpd",
               "sudo systemctl start httpd",

    ]
  }

}
4. Launch one Volume (EBS) and mount that volume into /var/www/html

//EBS creation

resource "aws_ebs_volume" "ebs1" {
  availability_zone = aws_instance.webserver.availability_zone
  size              = 1

  tags = {
    Name = "myhdterraform"
  }
}

// ebs attachment

resource "aws_volume_attachment" "ebs_att" {
  device_name = "/dev/sdk"
  volume_id   =  aws_ebs_volume.ebs1.id
  instance_id =  aws_instance.webserver.id
  force_detach = true
}
5. Developer have uploded the code into github repo also the repo has some images.
6. Copy the github repo code into /var/www/html
//connection to git hub repo with instance and copy the wepages
resource "null_resource" "connect_hd"{
 depends_on = [
       aws_volume_attachment.ebs_att
    ]
connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key  = file("C:/Users/hello/Downloads/mykey3333.pem")
    host     = aws_instance.webserver.public_ip 
  }
 
 
 provisioner "remote-exec" {
    inline = [ 
               "sudo mkfs.ext4 /dev/xvdk",
               "sudo mount /dev/xvdk  /var/www/html",
               "sudo rm -rf /var/www/html/*",
               "sudo git clone https://github.com/vmukul41/cloudterraform.git /var/www/html/"
      
               
      
    ]
  }
}

7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.
// s3 bucket creation and uploading a file to that bucket 

resource "aws_s3_bucket" "terraformcreation" {
depends_on = [
       null_resource.connect_hd
    ]

  bucket = "testing111222333"
  acl    = "public-read"
  region = "ap-south-1"
  tags = {
    Name        = "Mynewbucket"
  }

}

resource "aws_s3_bucket_object" "examplebucket_object" {
  key        = "mukul.jpg"
  bucket     =  aws_s3_bucket.terraformcreation.id
  source     = "C:/Users/hello/Desktop/mukul.jpg"
  acl    = "public-read"
}



8 Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to  update in code in /var/www/html
// cloud front creation and assign the origin from s3 bucket 

resource "aws_cloudfront_distribution" "s3_distribution" {
 
 depends_on = [
       aws_s3_bucket_object.examplebucket_object
    ]

  origin {
    domain_name = "testing111222333.s3.amazonaws.com"
    origin_id   = "S3-testing111222333"
    
  }
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "trying couldfront"
  default_root_object = "mukul.jpg"


default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-testing111222333"

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    viewer_protocol_policy = "allow-all"
  }
 
 price_class = "PriceClass_All"
 
  restrictions {
     geo_restriction {
      restriction_type = "none"
      
    }
  }

 tags = {
    Environment = "production"
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

connection {
    type     = "ssh"
    user     = "ec2-user"
    private_key  = file("C:/Users/hello/Downloads/mykey3333.pem")
    host     = aws_instance.webserver.public_ip 
  }

provisioner "remote-exec" {
    inline = [
      "sudo su <<END",
      "echo \"<img src='http://${aws_cloudfront_distribution.s3_distribution.domain_name}/${aws_s3_bucket_object.examplebucket_object.key}' height='200' width='200'>\" >> /var/www/html/index.php",
      "END",
    ]
  }



}

