#
AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 3 – REFACTORING
-
- This session of the configuration involves creating a backend where the tfstate file can be stored so that everybody in the team can securely have access to it.
- Create a file and name it backend.tf and populate it with the following code using the name of the bucket created in project 16.
```
# Note: The bucket name may not work for you since buckets are unique globally in AWS, so you must give it a unique name.
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terrafor-bucket"
  # Enable versioning so we can see the full revision history of our state files
  versioning {
    enabled = true
  }
  # Enable server-side encryption by default
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
```
- Create a DynamoDB table to handle locks and perform consistency checks.
```
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

- Before instructing terraform to make use of our S3 bucket as the backend to store the tfstate file, the DynamoDB must be provisioned. Run terraform apply now to do that.
- After provisioning the DynamoDB, paste the following code in backend.tf.
```
terraform {
  backend "s3" {
    bucket         = "dev-terraform-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```
-Run teraform init and verify the changes.
- The S3 bucket now has the tfstate file
![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/02ae7e3dedf8a50aaeb49f8ff78af8f789fa0780/README.md)

- After running terraform plan, navigate to the DynamoDB console to see the lockfile.
![](https://github.com/akinolafusi/DAREY-PBL-PROJECTS3/blob/4c525df2fbf6a3508326dab162031d878c31bd05/P18/dynamodb%20terra%20lock.PNG)

- Before running terraform apply, create a file output.tf so that the S3 bucket Amazon Resource Names ARN and DynamoDB table name can be displayed.
Paste this:
```
output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
  description = "The ARN of the S3 bucket"
}
output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_locks.name
  description = "The name of the DynamoDB table"
}
```

- Now run terraform apply. There should be 2 versions of the tfstate file in the S3 bucket now.

##
Security Groups refactoring with dynamic block
- 
-  For repitive block of code, dynamic block can be used. for example
```
# This long repitive security group code can be declared once and used multiple times

resource "aws_security_group" "ext-alb-sg" {
  name        = "ext-alb-sg"
  vpc_id      = aws_vpc.main.id
  description = "Allow TLS inbound traffic"

 ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

#Dynamic block format of the same code;

resource "aws_security_group" "ext-alb-sg" {
  name        = "ext-alb-sg"
  vpc_id      = aws_vpc.main.id
  description = "Allow TLS inbound traffic"

  dynamic "ingress" {
    for_each = local.ingress_ext-alb-sg
    content {
      from_port = ingress.value
      to_port   = ingress.value
      protocol  = "tcp"
      cidr_blocks = [ "0.0.0.0/0" ]
    }
  }

  dynamic "egress" {
    for_each = local.egress_ext-alb-sg
    content {
      from_port = egress.value
      to_port = egress.value
      protocol = "-1"
      cidr_blocks = [ "0.0.0.0/0"]
    }
  }
  tags = merge(
    var.tags,
    {
      Name = "ext-alb-sg"
    },
  )

}
```
##
EC2 refactoring with Map and Lookup
- 
- Map uses a key and value pairs as a data structure that can be set as a default type for variables.
```
variable "images" {
    type = "map"
    default = {
        us-east-1 = "image-1234"
        us-west-2 = "image-23834"
    }
}
```
To select the suiable AMI, the lookup function will be used:
```
resource "aws_instace" "web" {
    ami  = "${lookup(var.images, var.region), "ami-12323"}
}
```
###
Conditional Expressions
-
- It is used to make some decision and choose some resource based on a condition
In general, the syntax is as following: condition ? true_val : false_val.
for example
```
resource "aws_db_instance" "read_replica" {
  count               = var.create_read_replica == true ? 1 : 0
  replicate_source_db = aws_db_instance.this.id
}


```

- true 
- #condition equals to ‘if true’
- ? #means, set to ‘1`
- : #means, otherwise, set to ‘0’
- 