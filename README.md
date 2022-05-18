<h1>Deploying Django to AWS ECS with Terraform</h1>

<p>Create a new project directory along with a new Django project:</p>
<pre><span></span><code>$ mkdir django-ecs-terraform <span class="o">&amp;&amp;</span> <span class="nb">cd</span> django-ecs-terraform
$ mkdir app <span class="o">&amp;&amp;</span> <span class="nb">cd</span> app
$ python3.10 -m venv env
$ <span class="nb">source</span> env/bin/activate

<span class="o">(</span>env<span class="o">)</span>$ pip install <span class="nv">django</span><span class="o">==</span><span class="m">3</span>.2.9
<span class="o">(</span>env<span class="o">)</span>$ django-admin startproject hello_django .
<span class="o">(</span>env<span class="o">)</span>$ python manage.py migrate
<span class="o">(</span>env<span class="o">)</span>$ python manage.py runserver
</code></pre>
<p>Navigate to <a href="http://localhost:8000/">http://localhost:8000/</a> to view the Django welcome screen. Kill the server once done, and then exit from the virtual environment. Go ahead and remove it as well. We now have a simple Django project to work with.</p>
<p>Add a <em>requirements.txt</em> file:</p>
<pre><span></span><code>Django==3.2.9
gunicorn==20.1.0
</code></pre>
<p>Add a <em>Dockerfile</em> as well:</p>
<pre><span></span><code><span class="c"># pull official base image</span>
<span class="k">FROM</span><span class="w"> </span><span class="s">python:3.9.0-slim-buster</span>

<span class="c"># set work directory</span>
<span class="k">WORKDIR</span><span class="w"> </span><span class="s">/usr/src/app</span>

<span class="c"># set environment variables</span>
<span class="k">ENV</span><span class="w"> </span>PYTHONDONTWRITEBYTECODE <span class="m">1</span>
<span class="k">ENV</span><span class="w"> </span>PYTHONUNBUFFERED <span class="m">1</span>

<span class="c"># install dependencies</span>
<span class="k">RUN</span><span class="w"> </span>pip install --upgrade pip
<span class="k">COPY</span><span class="w"> </span>./requirements.txt .
<span class="k">RUN</span><span class="w"> </span>pip install -r requirements.txt

<span class="c"># copy project</span>
<span class="k">COPY</span><span class="w"> </span>. .
</code></pre>
<p>For testing purposes, set <code>DEBUG</code> to <code>True</code> and allow all hosts in the <em>settings.py</em> file:</p>
<pre><span></span><code><span class="n">DEBUG</span> <span class="o">=</span> <span class="kc">True</span>

<span class="n">ALLOWED_HOSTS</span> <span class="o">=</span> <span class="p">[</span><span class="s1">'*'</span><span class="p">]</span>
</code></pre>
<p>Next, build and tag the image and spin up a new container:</p>
<pre><span></span><code>$ docker build -t django-ecs .

$ docker run <span class="se">\</span>
-p <span class="m">8007</span>:8000 <span class="se">\</span>
--name django-test <span class="se">\</span>
django-ecs <span class="se">\</span>
gunicorn hello_django.wsgi:application --bind <span class="m">0</span>.0.0.0:8000
</code></pre>
<p>Ensure you can view the welcome screen again at <a href="http://localhost:8007/">http://localhost:8007/</a>.</p>
<p>Stop and remove the container once done:</p>
<pre><span></span><code>$ docker stop django-test
$ docker rm django-test
</code></pre>
<p>Add a <em>.gitignore</em> file to the project root:</p>
<pre><span></span><code><span class="nf">__pycache__</span><span class="w"></span>
<span class="na">.DS_Store</span><span class="w"></span>
<span class="err">*</span><span class="na">.sqlite3</span><span class="w"></span>
</code></pre>
<p>Your project structure should now look like this:</p>
<pre><span></span><code>├── .gitignore
└── app
├── Dockerfile
├── hello_django
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── manage.py
└── requirements.txt
</code></pre>
<p>For a more detailed look at how to containerize a Django app, review the <a href="/blog/dockerizing-django-with-postgres-gunicorn-and-nginx/">Dockerizing Django with Postgres, Gunicorn, and Nginx</a> blog post.</p>
<h2 id="ecr">ECR</h2>
<p>Before jumping into Terraform, let's push the Docker image to <a href="https://aws.amazon.com/ecr/">Elastic Container Registry</a> (ECR), a private Docker image registry.</p>
<p>Navigate to the <a href="http://console.aws.amazon.com/ecr">ECR console</a>, and add a new repository called "django-app". Keep the tags mutable. For more on this, review the <a href="https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-tag-mutability.html">Image Tag Mutability</a> guide.</p>
<p>Back in your terminal, build and tag the image again:</p>
<pre><span></span><code>$ docker build -t &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest .
</code></pre>
<p>Be sure to replace <code>&lt;AWS_ACCOUNT_ID&gt;</code> with your AWS account ID.</p>
<p>We'll be using the <code>us-west-1</code> region throughout this course. Feel free to change this if you'd like.</p>
<p>Authenticate the Docker CLI to use the ECR registry:</p>
<pre><span></span><code>$ aws ecr get-login --region us-west-1 --no-include-email
</code></pre>
<p>This command will provide an auth token. Copy and paste the entire <code>docker login</code> command to authenticate.</p>
<p>Push the image:</p>
<pre><span></span><code>$ docker push &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest
</code></pre>
<h2 id="terraform-setup">Terraform Setup</h2>
<p>Add a "terraform" folder to your project's root. We'll add each of our Terraform configuration files to this folder.</p>
<p>Next, add a new file to "terraform" called <em>01_provider.tf</em>:</p>
<pre><span></span><code>provider "aws" {
region = var.region
}
</code></pre>
<p>Here, we defined the <a href="https://registry.terraform.io/providers/hashicorp/aws/latest/docs">AWS</a> provider. You'll need to provide your AWS credentials in order to <a href="https://registry.terraform.io/providers/hashicorp/aws/latest/docs#authentication">authenticate</a>. Define them as environment variables:</p>
<pre><span></span><code>$ <span class="nb">export</span> <span class="nv">AWS_ACCESS_KEY_ID</span><span class="o">=</span><span class="s2">"YOUR_AWS_ACCESS_KEY_ID"</span>
$ <span class="nb">export</span> <span class="nv">AWS_SECRET_ACCESS_KEY</span><span class="o">=</span><span class="s2">"YOUR_AWS_SECRET_ACCESS_KEY"</span>
</code></pre>
<p>We used a string interpolated value for <code>region</code>, which will be read in from a <em>variables.tf</em> file. Go ahead and add this file to the "terraform" folder and add the following variable to it:</p>
<pre><span></span><code># core

variable "region" {
description = "The AWS region to create resources in."
default     = "us-west-1"
}
</code></pre>
<p>Feel free to update the variables as you go through this tutorial based on your specific requirements.</p>
<p>Run <code>terraform init</code> to create a new Terraform working directory and download the AWS provider.</p>
<p>With that we can start defining each piece of the AWS infrastructure.</p>
<h2 id="aws-resources">AWS Resources</h2>
<p>Next, let's configure the following AWS resources:</p>
<li>Networking:<ul>
<li>VPC</li>
<li>Public and private subnets</li>
<li>Routing tables</li>
<li>Internet Gateway</li>
<li>Key Pairs</li>
</ul>
</li>
<li>VPC</li>
<li>Public and private subnets</li>
<li>Routing tables</li>
<li>Internet Gateway</li>
<li>Key Pairs</li>
<li>Security Groups</li>
<li>Load Balancers, Listeners, and Target Groups</li>
<li>IAM Roles and Policies</li>
<li>ECS:<ul>
<li>Task Definition (with multiple containers)</li>
<li>Cluster</li>
<li>Service</li>
</ul>
</li>
<li>Task Definition (with multiple containers)</li>
<li>Cluster</li>
<li>Service</li>
<li>Launch Config and Auto Scaling Group</li>
<li>Health Checks and Logs</li>
<p> Terraform configuration files:  <a href="https://github.com/ShihabYasin/django-amazon-ecs-with-terraform">django-ecs-terraform</a> repo on GitHub.</p>
<h3 id="network-resources">Network Resources</h3>
<p>Let's define our network resources in a new file called <em>02_network.tf</em>:</p>
<pre><span></span><code># Production VPC
resource "aws_vpc" "production-vpc" {
cidr_block           = "10.0.0.0/16"
enable_dns_support   = true
enable_dns_hostnames = true
}

// Public subnets
resource "aws_subnet" "public-subnet-1" {
cidr_block        = var.public_subnet_1_cidr
vpc_id            = aws_vpc.production-vpc.id
availability_zone = var.availability_zones[0]
}
resource "aws_subnet" "public-subnet-2" {
cidr_block        = var.public_subnet_2_cidr
vpc_id            = aws_vpc.production-vpc.id
availability_zone = var.availability_zones[1]
}

// Private subnets
resource "aws_subnet" "private-subnet-1" {
cidr_block        = var.private_subnet_1_cidr
vpc_id            = aws_vpc.production-vpc.id
availability_zone = var.availability_zones[0]
}
resource "aws_subnet" "private-subnet-2" {
cidr_block        = var.private_subnet_2_cidr
vpc_id            = aws_vpc.production-vpc.id
availability_zone = var.availability_zones[1]
}

// Route tables for the subnets
resource "aws_route_table" "public-route-table" {
vpc_id = aws_vpc.production-vpc.id
}
resource "aws_route_table" "private-route-table" {
vpc_id = aws_vpc.production-vpc.id
}

// Associate the newly created route tables to the subnets
resource "aws_route_table_association" "public-route-1-association" {
route_table_id = aws_route_table.public-route-table.id
subnet_id      = aws_subnet.public-subnet-1.id
}
resource "aws_route_table_association" "public-route-2-association" {
route_table_id = aws_route_table.public-route-table.id
subnet_id      = aws_subnet.public-subnet-2.id
}
resource "aws_route_table_association" "private-route-1-association" {
route_table_id = aws_route_table.private-route-table.id
subnet_id      = aws_subnet.private-subnet-1.id
}
resource "aws_route_table_association" "private-route-2-association" {
route_table_id = aws_route_table.private-route-table.id
subnet_id      = aws_subnet.private-subnet-2.id
}

// Elastic IP
resource "aws_eip" "elastic-ip-for-nat-gw" {
vpc                       = true
associate_with_private_ip = "10.0.0.5"
depends_on                = [aws_internet_gateway.production-igw]
}

// NAT gateway
resource "aws_nat_gateway" "nat-gw" {
allocation_id = aws_eip.elastic-ip-for-nat-gw.id
subnet_id     = aws_subnet.public-subnet-1.id
depends_on    = [aws_eip.elastic-ip-for-nat-gw]
}
resource "aws_route" "nat-gw-route" {
route_table_id         = aws_route_table.private-route-table.id
nat_gateway_id         = aws_nat_gateway.nat-gw.id
destination_cidr_block = "0.0.0.0/0"
}

// Internet Gateway for the public subnet
resource "aws_internet_gateway" "production-igw" {
vpc_id = aws_vpc.production-vpc.id
}

// Route the public subnet traffic through the Internet Gateway
resource "aws_route" "public-internet-igw-route" {
route_table_id         = aws_route_table.public-route-table.id
gateway_id             = aws_internet_gateway.production-igw.id
destination_cidr_block = "0.0.0.0/0"
}
</code></pre>
<p>Here, we defined the following resources:</p>
<ol>
<li><a href="https://aws.amazon.com/vpc/">Virtual Private Cloud</a> (VPC)</li>
<li><a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html">Public and private subnets</a></li>
<li><a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html">Route tables</a></li>
<li><a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html">Internet Gateway</a></li>
</ol>
<li><a href="https://aws.amazon.com/vpc/">Virtual Private Cloud</a> (VPC)</li>
<li><a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html">Public and private subnets</a></li>
<li><a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html">Route tables</a></li>
<li><a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html">Internet Gateway</a></li>
<p>Add the following variables as well:</p>
<pre><span></span><code># networking

variable "public_subnet_1_cidr" {
description = "CIDR Block for Public Subnet 1"
default     = "10.0.1.0/24"
}
variable "public_subnet_2_cidr" {
description = "CIDR Block for Public Subnet 2"
default     = "10.0.2.0/24"
}
variable "private_subnet_1_cidr" {
description = "CIDR Block for Private Subnet 1"
default     = "10.0.3.0/24"
}
variable "private_subnet_2_cidr" {
description = "CIDR Block for Private Subnet 2"
default     = "10.0.4.0/24"
}
variable "availability_zones" {
description = "Availability zones"
type        = list(string)
default     = ["us-west-1b", "us-west-1c"]
}
</code></pre>
<p>Run <code>terraform plan</code> to generate and show the execution plan based on the defined configuration.</p>
<h3 id="security-groups">Security Groups</h3>
<p>Moving on, to protect the Django app and ECS cluster, let's configure <a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html">Security Groups</a> in a new file called <em>03_securitygroups.tf</em>:</p>
<pre><span></span><code># ALB Security Group (Traffic Internet -&gt; ALB)
resource "aws_security_group" "load-balancer" {
name        = "load_balancer_security_group"
description = "Controls access to the ALB"
vpc_id      = aws_vpc.production-vpc.id

ingress {
from_port   = 80
to_port     = 80
protocol    = "tcp"
cidr_blocks = ["0.0.0.0/0"]
}

ingress {
from_port   = 443
to_port     = 443
protocol    = "tcp"
cidr_blocks = ["0.0.0.0/0"]
}

egress {
from_port   = 0
to_port     = 0
protocol    = "-1"
cidr_blocks = ["0.0.0.0/0"]
}
}

// ECS Security group (traffic ALB -&gt; ECS, ssh -&gt; ECS)
resource "aws_security_group" "ecs" {
name        = "ecs_security_group"
description = "Allows inbound access from the ALB only"
vpc_id      = aws_vpc.production-vpc.id

ingress {
from_port       = 0
to_port         = 0
protocol        = "-1"
security_groups = [aws_security_group.load-balancer.id]
}

ingress {
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
}
</code></pre>
<p>Take note of the <a href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html#SecurityGroupRules">inbound rule</a> on the Security Group associated with the ECS cluster for port 22. This is so we can SSH into an EC2 instance to run the initial DB migrations and add a super user.</p>
<h3 id="load-balancer">Load Balancer</h3>
<p>Next, let's configure an <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html">Application Load Balancer</a> (ALB) along with the appropriate <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html">Target Group</a> and <a href="https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html">Listener</a>.</p>
<p><em>04_loadbalancer.tf</em>:</p>
<pre><span></span><code># Production Load Balancer
resource "aws_lb" "production" {
name               = "${var.ecs_cluster_name}-alb"
load_balancer_type = "application"
internal           = false
security_groups    = [aws_security_group.load-balancer.id]
subnets            = [aws_subnet.public-subnet-1.id, aws_subnet.public-subnet-2.id]
}

// Target group
resource "aws_alb_target_group" "default-target-group" {
name     = "${var.ecs_cluster_name}-tg"
port     = 80
protocol = "HTTP"
vpc_id   = aws_vpc.production-vpc.id

health_check {
path                = var.health_check_path
port                = "traffic-port"
healthy_threshold   = 5
unhealthy_threshold = 2
timeout             = 2
interval            = 5
matcher             = "200"
}
}

// Listener (redirects traffic from the load balancer to the target group)
resource "aws_alb_listener" "ecs-alb-http-listener" {
load_balancer_arn = aws_lb.production.id
port              = "80"
protocol          = "HTTP"
depends_on        = [aws_alb_target_group.default-target-group]

default_action {
type             = "forward"
target_group_arn = aws_alb_target_group.default-target-group.arn
}
}
</code></pre>
<p>Add the required variables:</p>
<pre><span></span><code># load balancer

variable "health_check_path" {
description = "Health check path for the default target group"
default     = "/ping/"
}


// ecs

variable "ecs_cluster_name" {
description = "Name of the ECS cluster"
default     = "production"
}
</code></pre>
<p>So, we configured our load balancer and listener to listen for HTTP requests on port 80. This is temporary. After we verify that our infrastructure and application are set up correctly, we'll update the load balancer to listen for HTTPS requests on port 443.</p>
<p>Take note of the path URL for the health check: <code>/ping/</code>.</p>
<h3 id="iam-roles">IAM Roles</h3>
<p><em>05_iam.tf</em>:</p>
<pre><span></span><code>resource "aws_iam_role" "ecs-host-role" {
name               = "ecs_host_role_prod"
assume_role_policy = file("policies/ecs-role.json")
}

resource "aws_iam_role_policy" "ecs-instance-role-policy" {
name   = "ecs_instance_role_policy"
policy = file("policies/ecs-instance-role-policy.json")
role   = aws_iam_role.ecs-host-role.id
}

resource "aws_iam_role" "ecs-service-role" {
name               = "ecs_service_role_prod"
assume_role_policy = file("policies/ecs-role.json")
}

resource "aws_iam_role_policy" "ecs-service-role-policy" {
name   = "ecs_service_role_policy"
policy = file("policies/ecs-service-role-policy.json")
role   = aws_iam_role.ecs-service-role.id
}

resource "aws_iam_instance_profile" "ecs" {
name = "ecs_instance_profile_prod"
path = "/"
role = aws_iam_role.ecs-host-role.name
}
</code></pre>
<p>Add a new folder in "terraform" called "policies". Then, add the following <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html">role</a> and <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html">policy</a> definitions:</p>
<p><em>ecs-role.json</em>:</p>
<pre><span></span><code><span class="p">{</span><span class="w"></span>
<span class="w">  </span><span class="nt">"Version"</span><span class="p">:</span><span class="w"> </span><span class="s2">"2008-10-17"</span><span class="p">,</span><span class="w"></span>
<span class="w">  </span><span class="nt">"Statement"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">    </span><span class="p">{</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Action"</span><span class="p">:</span><span class="w"> </span><span class="s2">"sts:AssumeRole"</span><span class="p">,</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Principal"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"Service"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">          </span><span class="s2">"ecs.amazonaws.com"</span><span class="p">,</span><span class="w"></span>
<span class="w">          </span><span class="s2">"ec2.amazonaws.com"</span><span class="w"></span>
<span class="w">        </span><span class="p">]</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Effect"</span><span class="p">:</span><span class="w"> </span><span class="s2">"Allow"</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>
<span class="w">  </span><span class="p">]</span><span class="w"></span>
<span class="p">}</span><span class="w"></span>
</code></pre>
<p><em>ecs-instance-role-policy.json</em>:</p>
<pre><span></span><code><span class="p">{</span><span class="w"></span>
<span class="w">  </span><span class="nt">"Version"</span><span class="p">:</span><span class="w"> </span><span class="s2">"2012-10-17"</span><span class="p">,</span><span class="w"></span>
<span class="w">  </span><span class="nt">"Statement"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">    </span><span class="p">{</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Effect"</span><span class="p">:</span><span class="w"> </span><span class="s2">"Allow"</span><span class="p">,</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Action"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">        </span><span class="s2">"ecs:*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"ec2:*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"elasticloadbalancing:*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"ecr:*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"cloudwatch:*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"s3:*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"rds:*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"logs:*"</span><span class="w"></span>
<span class="w">      </span><span class="p">],</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Resource"</span><span class="p">:</span><span class="w"> </span><span class="s2">"*"</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>
<span class="w">  </span><span class="p">]</span><span class="w"></span>
<span class="p">}</span><span class="w"></span>
</code></pre>
<p><em>ecs-service-role-policy.json</em>:</p>
<pre><span></span><code><span class="p">{</span><span class="w"></span>
<span class="w">  </span><span class="nt">"Version"</span><span class="p">:</span><span class="w"> </span><span class="s2">"2012-10-17"</span><span class="p">,</span><span class="w"></span>
<span class="w">  </span><span class="nt">"Statement"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">    </span><span class="p">{</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Effect"</span><span class="p">:</span><span class="w"> </span><span class="s2">"Allow"</span><span class="p">,</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Action"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">        </span><span class="s2">"elasticloadbalancing:Describe*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"elasticloadbalancing:DeregisterInstancesFromLoadBalancer"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"elasticloadbalancing:RegisterInstancesWithLoadBalancer"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"ec2:Describe*"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"ec2:AuthorizeSecurityGroupIngress"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"elasticloadbalancing:RegisterTargets"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="s2">"elasticloadbalancing:DeregisterTargets"</span><span class="w"></span>
<span class="w">      </span><span class="p">],</span><span class="w"></span>
<span class="w">      </span><span class="nt">"Resource"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">        </span><span class="s2">"*"</span><span class="w"></span>
<span class="w">      </span><span class="p">]</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>
<span class="w">  </span><span class="p">]</span><span class="w"></span>
<span class="p">}</span><span class="w"></span>
</code></pre>
<h3 id="logs">Logs</h3>
<p><em>06_logs.tf</em>:</p>
<pre><span></span><code>resource "aws_cloudwatch_log_group" "django-log-group" {
name              = "/ecs/django-app"
retention_in_days = var.log_retention_in_days
}

resource "aws_cloudwatch_log_stream" "django-log-stream" {
name           = "django-app-log-stream"
log_group_name = aws_cloudwatch_log_group.django-log-group.name
}
</code></pre>
<p>Add the variable:</p>
<pre><span></span><code># logs

variable "log_retention_in_days" {
default = 30
}
</code></pre>
<h3 id="key-pair">Key Pair</h3>
<p><em>07_keypair.tf</em>:</p>
<pre><span></span><code>resource "aws_key_pair" "production" {
key_name   = "${var.ecs_cluster_name}_key_pair"
public_key = file(var.ssh_pubkey_file)
}
</code></pre>
<p>Variable:</p>
<pre><span></span><code># key pair

variable "ssh_pubkey_file" {
description = "Path to an SSH public key"
default     = "~/.ssh/id_rsa.pub"
}
</code></pre>
<h3 id="ecs">ECS</h3>
<p>Now, we can configure our <a href="https://aws.amazon.com/ecs/">ECS</a> cluster.</p>
<p><em>08_ecs.tf</em>:</p>
<pre><span></span><code>resource "aws_ecs_cluster" "production" {
name = "${var.ecs_cluster_name}-cluster"
}

resource "aws_launch_configuration" "ecs" {
name                        = "${var.ecs_cluster_name}-cluster"
image_id                    = lookup(var.amis, var.region)
instance_type               = var.instance_type
security_groups             = [aws_security_group.ecs.id]
iam_instance_profile        = aws_iam_instance_profile.ecs.name
key_name                    = aws_key_pair.production.key_name
associate_public_ip_address = true
user_data                   = "#!/bin/bash\necho ECS_CLUSTER='${var.ecs_cluster_name}-cluster' &gt; /etc/ecs/ecs.config"
}

data "template_file" "app" {
template = file("templates/django_app.json.tpl")

vars = {
docker_image_url_django = var.docker_image_url_django
region                  = var.region
}
}

resource "aws_ecs_task_definition" "app" {
family                = "django-app"
container_definitions = data.template_file.app.rendered
}

resource "aws_ecs_service" "production" {
name            = "${var.ecs_cluster_name}-service"
cluster         = aws_ecs_cluster.production.id
task_definition = aws_ecs_task_definition.app.arn
iam_role        = aws_iam_role.ecs-service-role.arn
desired_count   = var.app_count
depends_on      = [aws_alb_listener.ecs-alb-http-listener, aws_iam_role_policy.ecs-service-role-policy]

load_balancer {
target_group_arn = aws_alb_target_group.default-target-group.arn
container_name   = "django-app"
container_port   = 8000
}
}
</code></pre>
<p>Take a look at the <code>user_data</code> field in the <code>aws_launch_configuration</code>. Put simply, <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html">user_data</a> is a script that is run when a new EC2 instance is launched. In order for the ECS cluster to discover new EC2 instances, the cluster name needs to be added to the <code>ECS_CLUSTER</code> environment variable within the <em>/etc/ecs/ecs.config</em> config file within the instance. In other words, the following script will run when a new instance is bootstrapped allowing it to be discovered by the cluster:</p>
<pre><span></span><code><span class="ch">#!/bin/bash</span>

<span class="nb">echo</span> <span class="nv">ECS_CLUSTER</span><span class="o">=</span><span class="s1">'production-cluster'</span> &gt; /etc/ecs/ecs.config
</code></pre>
<p>For more on this discovery process, check out <a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-agent-config.html">Amazon ECS Container Agent Configuration</a> guide.</p>
<p>Add a "templates" folder to the "terraform" folder, and then add a new template file called <em>django_app.json.tpl</em>:</p>
<pre><span></span><code><span class="p">[</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"django-app"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"image"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${docker_image_url_django}"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"essential"</span><span class="p">:</span><span class="w"> </span><span class="kc">true</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"cpu"</span><span class="p">:</span><span class="w"> </span><span class="mi">10</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"memory"</span><span class="p">:</span><span class="w"> </span><span class="mi">512</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"links"</span><span class="p">:</span><span class="w"> </span><span class="p">[],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"portMappings"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"containerPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">8000</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"hostPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">0</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"protocol"</span><span class="p">:</span><span class="w"> </span><span class="s2">"tcp"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"command"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"gunicorn"</span><span class="p">,</span><span class="w"> </span><span class="s2">"-w"</span><span class="p">,</span><span class="w"> </span><span class="s2">"3"</span><span class="p">,</span><span class="w"> </span><span class="s2">"-b"</span><span class="p">,</span><span class="w"> </span><span class="s2">":8000"</span><span class="p">,</span><span class="w"> </span><span class="s2">"hello_django.wsgi:application"</span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"environment"</span><span class="p">:</span><span class="w"> </span><span class="p">[],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"logConfiguration"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">      </span><span class="nt">"logDriver"</span><span class="p">:</span><span class="w"> </span><span class="s2">"awslogs"</span><span class="p">,</span><span class="w"></span>
<span class="w">      </span><span class="nt">"options"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-group"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/ecs/django-app"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-region"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${region}"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-stream-prefix"</span><span class="p">:</span><span class="w"> </span><span class="s2">"django-app-log-stream"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>
<span class="w">  </span><span class="p">}</span><span class="w"></span>
<span class="p">]</span><span class="w"></span>
</code></pre>
<p>Here, we defined our <a href="https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ContainerDefinition.html">container definition</a> associated with the Django app.</p>
<p>Add the following variables as well:</p>
<pre><span></span><code># ecs

variable "ecs_cluster_name" {
description = "Name of the ECS cluster"
default     = "production"
}
variable "amis" {
description = "Which AMI to spawn."
default = {
us-west-1 = "ami-0bd3976c0dbacc605"
}
}
variable "instance_type" {
default = "t2.micro"
}
variable "docker_image_url_django" {
description = "Docker image to run in the ECS cluster"
default     = "&lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest"
}
variable "app_count" {
description = "Number of Docker containers to run"
default     = 2
}
</code></pre>
<p>Again, be sure to replace <code>&lt;AWS_ACCOUNT_ID&gt;</code> with your AWS account ID.</p>
<p>Refer to the <a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html#ecs-optimized-ami-linux">Linux Amazon ECS-optimized AMIs</a> guide to find a list of AMIs with Docker pre-installed.</p>
<p>Since we added the <a href="https://registry.terraform.io/providers/hashicorp/template/latest/docs">Template</a> provider, run <code>terraform init</code> again to download the new provider.</p>
<h3 id="auto-scaling">Auto Scaling</h3>
<p><em>09_auto_scaling.tf</em>:</p>
<pre><span></span><code>resource "aws_autoscaling_group" "ecs-cluster" {
name                 = "${var.ecs_cluster_name}_auto_scaling_group"
min_size             = var.autoscale_min
max_size             = var.autoscale_max
desired_capacity     = var.autoscale_desired
health_check_type    = "EC2"
launch_configuration = aws_launch_configuration.ecs.name
vpc_zone_identifier  = [aws_subnet.private-subnet-1.id, aws_subnet.private-subnet-2.id]
}
</code></pre>
<p>New variables:</p>
<pre><span></span><code># auto scaling

variable "autoscale_min" {
description = "Minimum autoscale (number of EC2)"
default     = "1"
}
variable "autoscale_max" {
description = "Maximum autoscale (number of EC2)"
default     = "10"
}
variable "autoscale_desired" {
description = "Desired autoscale (number of EC2)"
default     = "4"
}
</code></pre>
<h3 id="test">Test</h3>
<p><em>outputs.tf</em>:</p>
<pre><span></span><code>output "alb_hostname" {
value = aws_lb.production.dns_name
}
</code></pre>
<p>Here, we configured an <em>outputs.tf</em> file along with an <a href="https://www.terraform.io/language/values/outputs">output value</a> called <code>alb_hostname</code>. After we execute the Terraform plan, to spin up the AWS infrastructure, the load balancer's DNS name will be outputted to the terminal.</p>
<p>Ready?!? View then execute the plan:</p>
<pre><span></span><code>$ terraform plan

$ terraform apply
</code></pre>
<p>You should see the health check failing with a 404:</p>
<pre><span></span><code>service production-service <span class="o">(</span>instance i-013f1192da079b0bf<span class="o">)</span> <span class="o">(</span>port <span class="m">49153</span><span class="o">)</span>
is unhealthy <span class="k">in</span> target-group production-tg due to
<span class="o">(</span>reason Health checks failed with these codes: <span class="o">[</span><span class="m">404</span><span class="o">])</span>
</code></pre>
<p>This is expected since we haven't set up a /ping/ handler in the app yet.</p>
<h2 id="django-health-check">Django Health Check</h2>
<p>Add the following middleware to <em>app/hello_django/middleware.py</em>:</p>
<pre><span></span><code><span class="kn">from</span> <span class="nn">django.http</span> <span class="kn">import</span> <span class="n">HttpResponse</span>
<span class="kn">from</span> <span class="nn">django.utils.deprecation</span> <span class="kn">import</span> <span class="n">MiddlewareMixin</span>


<span class="k">class</span> <span class="nc">HealthCheckMiddleware</span><span class="p">(</span><span class="n">MiddlewareMixin</span><span class="p">):</span>
<span class="k">def</span> <span class="nf">process_request</span><span class="p">(</span><span class="bp">self</span><span class="p">,</span> <span class="n">request</span><span class="p">):</span>
<span class="k">if</span> <span class="n">request</span><span class="o">.</span><span class="n">META</span><span class="p">[</span><span class="s1">'PATH_INFO'</span><span class="p">]</span> <span class="o">==</span> <span class="s1">'/ping/'</span><span class="p">:</span>
<span class="k">return</span> <span class="n">HttpResponse</span><span class="p">(</span><span class="s1">'pong!'</span><span class="p">)</span>
</code></pre>
<p>Add the class to the middleware config in <em>settings.py</em>:</p>
<pre><span></span><code><span class="n">MIDDLEWARE</span> <span class="o">=</span> <span class="p">[</span>
<span class="s1">'hello_django.middleware.HealthCheckMiddleware'</span><span class="p">,</span>  <span class="c1"># new</span>
<span class="s1">'django.middleware.security.SecurityMiddleware'</span><span class="p">,</span>
<span class="s1">'django.contrib.sessions.middleware.SessionMiddleware'</span><span class="p">,</span>
<span class="s1">'django.middleware.common.CommonMiddleware'</span><span class="p">,</span>
<span class="s1">'django.middleware.csrf.CsrfViewMiddleware'</span><span class="p">,</span>
<span class="s1">'django.contrib.auth.middleware.AuthenticationMiddleware'</span><span class="p">,</span>
<span class="s1">'django.contrib.messages.middleware.MessageMiddleware'</span><span class="p">,</span>
<span class="s1">'django.middleware.clickjacking.XFrameOptionsMiddleware'</span><span class="p">,</span>
<span class="p">]</span>
</code></pre>
<p>This middleware is used to handle requests to the /ping/ URL before <code>ALLOWED_HOSTS</code> is checked. Why is this necessary?</p>
<p>The health check request comes from the EC2 instance. Since we don't know the private IP beforehand, this will ensure that the /ping/ route always returns a successful response even after we restrict <code>ALLOWED_HOSTS</code>.</p>
<p>It's worth noting that you could toss Nginx in front of Gunicorn and handle the health check in the Nginx config like so:</p>
<pre><span></span><span class="err">location /ping/ {</span>
<span class="err">    access_log off;</span>
<span class="err">    return 200;</span>
<span class="err">}</span>
</pre>
<p>To test locally, build the new image and then spin up the container:</p>
<pre><span></span><code>$ docker build -t django-ecs .

$ docker run <span class="se">\</span>
-p <span class="m">8007</span>:8000 <span class="se">\</span>
--name django-test <span class="se">\</span>
django-ecs <span class="se">\</span>
gunicorn hello_django.wsgi:application --bind <span class="m">0</span>.0.0.0:8000
</code></pre>
<p>Make sure <a href="http://localhost:8007/ping/">http://localhost:8007/ping/</a> works as expected:</p>
<pre><span></span><code>pong!
</code></pre>
<p>Stop and remove the container once done:</p>
<pre><span></span><code>$ docker stop django-test
$ docker rm django-test
</code></pre>
<p>Next, update ECR:</p>
<pre><span></span><code>$ docker build -t &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest .
$ docker push &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest
</code></pre>
<p>Let's add a quick script to update the Task Definition and Service so that the new Tasks use the new image that we just pushed.</p>
<p>Create a "deploy" folder in the project root. Then, add an <em>update-ecs.py</em> file to that newly created folder:</p>
<pre><span></span><code><span class="kn">import</span> <span class="nn">boto3</span>
<span class="kn">import</span> <span class="nn">click</span>


<span class="k">def</span> <span class="nf">get_current_task_definition</span><span class="p">(</span><span class="n">client</span><span class="p">,</span> <span class="n">cluster</span><span class="p">,</span> <span class="n">service</span><span class="p">):</span>
<span class="n">response</span> <span class="o">=</span> <span class="n">client</span><span class="o">.</span><span class="n">describe_services</span><span class="p">(</span><span class="n">cluster</span><span class="o">=</span><span class="n">cluster</span><span class="p">,</span> <span class="n">services</span><span class="o">=</span><span class="p">[</span><span class="n">service</span><span class="p">])</span>
<span class="n">current_task_arn</span> <span class="o">=</span> <span class="n">response</span><span class="p">[</span><span class="s2">"services"</span><span class="p">][</span><span class="mi">0</span><span class="p">][</span><span class="s2">"taskDefinition"</span><span class="p">]</span>

<span class="n">response</span> <span class="o">=</span> <span class="n">client</span><span class="o">.</span><span class="n">describe_task_definition</span><span class="p">(</span><span class="n">taskDefinition</span><span class="o">=</span><span class="n">current_task_arn</span><span class="p">)</span>
<span class="k">return</span> <span class="n">response</span>


<span class="nd">@click</span><span class="o">.</span><span class="n">command</span><span class="p">()</span>
<span class="nd">@click</span><span class="o">.</span><span class="n">option</span><span class="p">(</span><span class="s2">"--cluster"</span><span class="p">,</span> <span class="n">help</span><span class="o">=</span><span class="s2">"Name of the ECS cluster"</span><span class="p">,</span> <span class="n">required</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
<span class="nd">@click</span><span class="o">.</span><span class="n">option</span><span class="p">(</span><span class="s2">"--service"</span><span class="p">,</span> <span class="n">help</span><span class="o">=</span><span class="s2">"Name of the ECS service"</span><span class="p">,</span> <span class="n">required</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
<span class="k">def</span> <span class="nf">deploy</span><span class="p">(</span><span class="n">cluster</span><span class="p">,</span> <span class="n">service</span><span class="p">):</span>
<span class="n">client</span> <span class="o">=</span> <span class="n">boto3</span><span class="o">.</span><span class="n">client</span><span class="p">(</span><span class="s2">"ecs"</span><span class="p">)</span>

<span class="n">response</span> <span class="o">=</span> <span class="n">get_current_task_definition</span><span class="p">(</span><span class="n">client</span><span class="p">,</span> <span class="n">cluster</span><span class="p">,</span> <span class="n">service</span><span class="p">)</span>
<span class="n">container_definition</span> <span class="o">=</span> <span class="n">response</span><span class="p">[</span><span class="s2">"taskDefinition"</span><span class="p">][</span><span class="s2">"containerDefinitions"</span><span class="p">][</span><span class="mi">0</span><span class="p">]</span><span class="o">.</span><span class="n">copy</span><span class="p">()</span>

<span class="n">response</span> <span class="o">=</span> <span class="n">client</span><span class="o">.</span><span class="n">register_task_definition</span><span class="p">(</span>
<span class="n">family</span><span class="o">=</span><span class="n">response</span><span class="p">[</span><span class="s2">"taskDefinition"</span><span class="p">][</span><span class="s2">"family"</span><span class="p">],</span>
<span class="n">volumes</span><span class="o">=</span><span class="n">response</span><span class="p">[</span><span class="s2">"taskDefinition"</span><span class="p">][</span><span class="s2">"volumes"</span><span class="p">],</span>
<span class="n">containerDefinitions</span><span class="o">=</span><span class="p">[</span><span class="n">container_definition</span><span class="p">],</span>
<span class="p">)</span>
<span class="n">new_task_arn</span> <span class="o">=</span> <span class="n">response</span><span class="p">[</span><span class="s2">"taskDefinition"</span><span class="p">][</span><span class="s2">"taskDefinitionArn"</span><span class="p">]</span>

<span class="n">response</span> <span class="o">=</span> <span class="n">client</span><span class="o">.</span><span class="n">update_service</span><span class="p">(</span>
<span class="n">cluster</span><span class="o">=</span><span class="n">cluster</span><span class="p">,</span> <span class="n">service</span><span class="o">=</span><span class="n">service</span><span class="p">,</span> <span class="n">taskDefinition</span><span class="o">=</span><span class="n">new_task_arn</span><span class="p">,</span>
<span class="p">)</span>


<span class="k">if</span> <span class="vm">__name__</span> <span class="o">==</span> <span class="s2">"__main__"</span><span class="p">:</span>
<span class="n">deploy</span><span class="p">()</span>
</code></pre>
<p>So, this script will create a new revision of the Task Definition and then update the Service so it uses the revised Task Definition.</p>
<p>Create and activate a new virtual environment. Then, install <a href="https://github.com/boto/boto3">Boto3</a> and <a href="https://click.palletsprojects.com/">Click</a>:</p>
<pre><span></span><code>$ pip install boto3 click
</code></pre>
<p>Add your AWS credentials along with the default region:</p>
<pre><span></span><code>$ <span class="nb">export</span> <span class="nv">AWS_ACCESS_KEY_ID</span><span class="o">=</span><span class="s2">"YOUR_AWS_ACCESS_KEY_ID"</span>
$ <span class="nb">export</span> <span class="nv">AWS_SECRET_ACCESS_KEY</span><span class="o">=</span><span class="s2">"YOUR_AWS_SECRET_ACCESS_KEY"</span>
$ <span class="nb">export</span> <span class="nv">AWS_DEFAULT_REGION</span><span class="o">=</span><span class="s2">"us-west-1"</span>
</code></pre>
<p>Run the script like so:</p>
<pre><span></span><code>$ python update-ecs.py --cluster<span class="o">=</span>production-cluster --service<span class="o">=</span>production-service
</code></pre>
<p>The Service should start two new Tasks based on the revised Task Definition and register them with the associated Target Group. This time the health checks should pass. You should now be able to view your application using the DNS hostname that was outputted to your terminal:</p>
<pre><span></span><code>Outputs:

<span class="nv">alb_hostname</span> <span class="o">=</span> production-alb-1008464563.us-west-1.elb.amazonaws.com
</code></pre>
<h2 id="rds">RDS</h2>
<p>Next, let's configure <a href="https://aws.amazon.com/rds/">RDS</a> so we can use Postgres for our production database.</p>
<p>Add a new Security Group to <em>03_securitygroups.tf</em> to ensure that only traffic from your ECS instance can talk to the database:</p>
<pre><span></span><code># RDS Security Group (traffic ECS -&gt; RDS)
resource "aws_security_group" "rds" {
name        = "rds-security-group"
description = "Allows inbound access from ECS only"
vpc_id      = aws_vpc.production-vpc.id

ingress {
protocol        = "tcp"
from_port       = "5432"
to_port         = "5432"
security_groups = [aws_security_group.ecs.id]
}

egress {
protocol    = "-1"
from_port   = 0
to_port     = 0
cidr_blocks = ["0.0.0.0/0"]
}
}
</code></pre>
<p>Next, add a new file called <em>10_rds.tf</em> for setting up the database itself:</p>
<pre><span></span><code>resource "aws_db_subnet_group" "production" {
name       = "main"
subnet_ids = [aws_subnet.private-subnet-1.id, aws_subnet.private-subnet-2.id]
}

resource "aws_db_instance" "production" {
identifier              = "production"
name                    = var.rds_db_name
username                = var.rds_username
password                = var.rds_password
port                    = "5432"
engine                  = "postgres"
engine_version          = "12.3"
instance_class          = var.rds_instance_class
allocated_storage       = "20"
storage_encrypted       = false
vpc_security_group_ids  = [aws_security_group.rds.id]
db_subnet_group_name    = aws_db_subnet_group.production.name
multi_az                = false
storage_type            = "gp2"
publicly_accessible     = false
backup_retention_period = 7
skip_final_snapshot     = true
}
</code></pre>
<p>Variables:</p>
<pre><span></span><code># rds

variable "rds_db_name" {
description = "RDS database name"
default     = "mydb"
}
variable "rds_username" {
description = "RDS database username"
default     = "foo"
}
variable "rds_password" {
description = "RDS database password"
}
variable "rds_instance_class" {
description = "RDS instance type"
default     = "db.t2.micro"
}
</code></pre>
<p>Note that we left off the default value for the password. More on this in a bit.</p>
<p>Since we'll need to know the address for the instance in our Django app, add a <code>depends_on</code> argument to the <code>aws_ecs_task_definition</code> in <em>08_ecs.tf</em>:</p>
<pre><span></span><code>resource "aws_ecs_task_definition" "app" {
family                = "django-app"
container_definitions = data.template_file.app.rendered
depends_on            = [aws_db_instance.production]
}
</code></pre>
<p>Next, we need to update the <code>DATABASES</code> config in <em>settings.py</em>:</p>
<pre><span></span><code><span class="k">if</span> <span class="s1">'RDS_DB_NAME'</span> <span class="ow">in</span> <span class="n">os</span><span class="o">.</span><span class="n">environ</span><span class="p">:</span>
<span class="n">DATABASES</span> <span class="o">=</span> <span class="p">{</span>
<span class="s1">'default'</span><span class="p">:</span> <span class="p">{</span>
<span class="s1">'ENGINE'</span><span class="p">:</span> <span class="s1">'django.db.backends.postgresql_psycopg2'</span><span class="p">,</span>
<span class="s1">'NAME'</span><span class="p">:</span> <span class="n">os</span><span class="o">.</span><span class="n">environ</span><span class="p">[</span><span class="s1">'RDS_DB_NAME'</span><span class="p">],</span>
<span class="s1">'USER'</span><span class="p">:</span> <span class="n">os</span><span class="o">.</span><span class="n">environ</span><span class="p">[</span><span class="s1">'RDS_USERNAME'</span><span class="p">],</span>
<span class="s1">'PASSWORD'</span><span class="p">:</span> <span class="n">os</span><span class="o">.</span><span class="n">environ</span><span class="p">[</span><span class="s1">'RDS_PASSWORD'</span><span class="p">],</span>
<span class="s1">'HOST'</span><span class="p">:</span> <span class="n">os</span><span class="o">.</span><span class="n">environ</span><span class="p">[</span><span class="s1">'RDS_HOSTNAME'</span><span class="p">],</span>
<span class="s1">'PORT'</span><span class="p">:</span> <span class="n">os</span><span class="o">.</span><span class="n">environ</span><span class="p">[</span><span class="s1">'RDS_PORT'</span><span class="p">],</span>
<span class="p">}</span>
<span class="p">}</span>
<span class="k">else</span><span class="p">:</span>
<span class="n">DATABASES</span> <span class="o">=</span> <span class="p">{</span>
<span class="s1">'default'</span><span class="p">:</span> <span class="p">{</span>
<span class="s1">'ENGINE'</span><span class="p">:</span> <span class="s1">'django.db.backends.sqlite3'</span><span class="p">,</span>
<span class="s1">'NAME'</span><span class="p">:</span> <span class="n">os</span><span class="o">.</span><span class="n">path</span><span class="o">.</span><span class="n">join</span><span class="p">(</span><span class="n">BASE_DIR</span><span class="p">,</span> <span class="s1">'db.sqlite3'</span><span class="p">),</span>
<span class="p">}</span>
<span class="p">}</span>
</code></pre>
<p>Add the import:</p>
<pre><span></span><code><span class="kn">import</span> <span class="nn">os</span>
</code></pre>
<p>Update the <code>environment</code> section in the <em>django_app.json.tpl</em> template:</p>
<pre><span></span><code><span class="nt">"environment"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_DB_NAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_db_name}"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_USERNAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_username}"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_PASSWORD"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_password}"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_HOSTNAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_hostname}"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_PORT"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"5432"</span><span class="w"></span>
<span class="w">  </span><span class="p">}</span><span class="w"></span>
<span class="p">],</span><span class="w"></span>
</code></pre>
<p>Update the vars passed to the template in <em>08_ecs.tf</em>:</p>
<pre><span></span><code>data "template_file" "app" {
template = file("templates/django_app.json.tpl")

vars = {
docker_image_url_django = var.docker_image_url_django
region                  = var.region
rds_db_name             = var.rds_db_name
rds_username            = var.rds_username
rds_password            = var.rds_password
rds_hostname            = aws_db_instance.production.address
}
}
</code></pre>
<p>Add <a href="https://www.psycopg.org/">Psycopg2</a> to the requirements file:</p>
<pre><span></span><code>Django==3.2.9
gunicorn==20.1.0
psycopg2-binary==2.9.2
</code></pre>
<p>Update the Dockerfile to install the appropriate packages required for Psycopg2:</p>
<pre><span></span><code><span class="c"># pull official base image</span>
<span class="k">FROM</span><span class="w"> </span><span class="s">python:3.9.0-slim-buster</span>

<span class="c"># set work directory</span>
<span class="k">WORKDIR</span><span class="w"> </span><span class="s">/usr/src/app</span>

<span class="c"># set environment variables</span>
<span class="k">ENV</span><span class="w"> </span>PYTHONDONTWRITEBYTECODE <span class="m">1</span>
<span class="k">ENV</span><span class="w"> </span>PYTHONUNBUFFERED <span class="m">1</span>

<span class="c"># install psycopg2 dependencies</span>
<span class="k">RUN</span><span class="w"> </span>apt-get update <span class="se">\</span>
<span class="o">&amp;&amp;</span> apt-get -y install gcc postgresql <span class="se">\</span>
<span class="o">&amp;&amp;</span> apt-get clean

<span class="c"># install dependencies</span>
<span class="k">RUN</span><span class="w"> </span>pip install --upgrade pip
<span class="k">COPY</span><span class="w"> </span>./requirements.txt .
<span class="k">RUN</span><span class="w"> </span>pip install -r requirements.txt

<span class="c"># copy project</span>
<span class="k">COPY</span><span class="w"> </span>. .
</code></pre>
<p>Alright. Build the Docker image and push it up to ECR. Then, to update the ECS Task Definition, create the RDS resources, and update the Service, run:</p>
<pre><span></span><code>$ terraform apply
</code></pre>
<p>Since we didn't set a default for the password, you'll be prompted to enter one:</p>
<pre><span></span><code>var.rds_password
RDS database password

Enter a value:
</code></pre>
<p>Rather than having to pass a value in each time, you could set an <a href="https://www.terraform.io/docs/commands/environment-variables.html#tf_var_name">environment variable</a> like so:</p>
<pre><span></span><code>$ <span class="nb">export</span> <span class="nv">TF_VAR_rds_password</span><span class="o">=</span>foobarbaz

$ terraform apply
</code></pre>
<p>Keep in mind that this approach, of using environment variables, keeps sensitive variables out of the <em>.tf</em> files, but they are still stored in the <em>terraform.tfstate</em> file in plain text. So, be sure to keep this file out of version control. Since keeping it out of version control doesn't work if other people on your team need access to it, look to either encrypting the secrets or using a secret store like <a href="https://www.vaultproject.io/">Vault</a> or <a href="https://aws.amazon.com/secrets-manager/">AWS Secrets Manager</a>.</p>
<p>After the new Tasks are registered with the Target Group, SSH into an EC2 instance where one of the Tasks is running:</p>
<pre><span></span><code>$ ssh <a class="__cf_email__" data-cfemail="5b3e3869762e283e291b" href="/cdn-cgi/l/email-protection">[email protected]</a>&lt;instance-ip&gt;

<span class="c1"># ssh <a class="__cf_email__" data-cfemail="a8cdcb9a85dddbcddae89d9a869d9b86999d9c8698" href="/cdn-cgi/l/email-protection">[email protected]</a></span>
</code></pre>
<p>Grab the container ID via <code>docker ps</code>, and use it to apply the migrations:</p>
<pre><span></span><code>$ docker <span class="nb">exec</span> -it &lt;container-id&gt; python manage.py migrate

<span class="c1"># docker exec -it 73284cda8a87 python manage.py migrate</span>
</code></pre>
<p>You may want to create a super user as well. Once done, exit from the SSH session. You'll probably want to remove the following inbound rule from the ECS Security group if you don't need SSH access any longer:</p>
<pre><span></span><code>ingress {
from_port   = 22
to_port     = 22
protocol    = "tcp"
cidr_blocks = ["0.0.0.0/0"]
}
</code></pre>
<h2 id="domain-and-ssl-certificate">Domain and SSL Certificate</h2>
<p>Assuming you've generated and validated a new SSL certificate from <a href="https://aws.amazon.com/certificate-manager/">AWS Certificate Manager</a>, add the certificate's ARN to your variables:</p>
<pre><span></span><code># domain

variable "certificate_arn" {
description = "AWS Certificate Manager ARN for validated domain"
default     = "ADD YOUR ARN HERE"
}
</code></pre>
<p>Update the default listener associated with the load balancer in <em>04_loadbalancer.tf</em> so that it listens for HTTPS requests on port 443 (as opposed to HTTP on port 80):</p>
<pre><span></span><code># Listener (redirects traffic from the load balancer to the target group)
resource "aws_alb_listener" "ecs-alb-http-listener" {
load_balancer_arn = aws_lb.production.id
port              = "443"
protocol          = "HTTPS"
ssl_policy        = "ELBSecurityPolicy-2016-08"
certificate_arn   = var.certificate_arn
depends_on        = [aws_alb_target_group.default-target-group]

default_action {
type             = "forward"
target_group_arn = aws_alb_target_group.default-target-group.arn
}
}
</code></pre>
<p>Apply the changes:</p>
<pre><span></span><code>$ terraform apply
</code></pre>
<p>Make sure to point your domain at the load balancer using a CNAME record. Make sure you can view your application.</p>
<h2 id="nginx">Nginx</h2>
<p>Next, let's add Nginx into the mix to handle requests for static files appropriately.</p>
<p>In the project root, create the following files and folders:</p>
<pre><span></span><code>└── nginx
├── Dockerfile
└── nginx.conf
</code></pre>
<p><em>Dockerfile</em>:</p>
<pre><span></span><code><span class="k">FROM</span><span class="w"> </span><span class="s">nginx:1.19.0-alpine</span>

<span class="k">RUN</span><span class="w"> </span>rm /etc/nginx/conf.d/default.conf
<span class="k">COPY</span><span class="w"> </span>nginx.conf /etc/nginx/conf.d
<span class="k">EXPOSE</span><span class="w"> </span><span class="s">80</span>
</code></pre>
<p><em>nginx.conf</em>:</p>
<pre><span></span><code><span class="nt">upstream</span><span class="w"> </span><span class="nt">hello_django</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="err">server</span><span class="w"> </span><span class="n">django-app</span><span class="p">:</span><span class="mi">8000</span><span class="p">;</span><span class="w"></span>
<span class="p">}</span><span class="w"></span>

<span class="nt">server</span><span class="w"> </span><span class="p">{</span><span class="w"></span>

<span class="w">    </span><span class="err">listen</span><span class="w"> </span><span class="err">80</span><span class="p">;</span><span class="w"></span>

<span class="w">    </span><span class="err">location</span><span class="w"> </span><span class="err">/</span><span class="w"> </span><span class="err">{</span><span class="w"></span>
<span class="w">        </span><span class="err">proxy_pass</span><span class="w"> </span><span class="n">http</span><span class="p">:</span><span class="o">//</span><span class="n">hello_django</span><span class="p">;</span><span class="w"></span>
<span class="w">        </span><span class="err">proxy_set_header</span><span class="w"> </span><span class="err">X-Forwarded-For</span><span class="w"> </span><span class="err">$proxy_add_x_forwarded_for</span><span class="p">;</span><span class="w"></span>
<span class="w">        </span><span class="err">proxy_set_header</span><span class="w"> </span><span class="err">Host</span><span class="w"> </span><span class="err">$host</span><span class="p">;</span><span class="w"></span>
<span class="w">        </span><span class="err">proxy_redirect</span><span class="w"> </span><span class="err">off</span><span class="p">;</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>

<span class="err">}</span><span class="w"></span>
</code></pre>
<p>Here, we set up a single location block, routing all traffic to the Django app. We'll set up a new location block for static files in the next section.</p>
<p>Create a new repo in ECR called "nginx", and then build and push the new image:</p>
<pre><span></span><code>$ docker build -t &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/nginx:latest .
$ docker push &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/nginx:latest
</code></pre>
<p>Add the following variable to the ECS section of the variables file:</p>
<pre><span></span><code>variable "docker_image_url_nginx" {
description = "Docker image to run in the ECS cluster"
default     = "&lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/nginx:latest"
}
</code></pre>
<p>Add the new container definition to the <em>django_app.json.tpl</em> template:</p>
<pre><span></span><code><span class="p">[</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"django-app"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"image"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${docker_image_url_django}"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"essential"</span><span class="p">:</span><span class="w"> </span><span class="kc">true</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"cpu"</span><span class="p">:</span><span class="w"> </span><span class="mi">10</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"memory"</span><span class="p">:</span><span class="w"> </span><span class="mi">512</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"links"</span><span class="p">:</span><span class="w"> </span><span class="p">[],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"portMappings"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"containerPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">8000</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"hostPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">0</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"protocol"</span><span class="p">:</span><span class="w"> </span><span class="s2">"tcp"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"command"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"gunicorn"</span><span class="p">,</span><span class="w"> </span><span class="s2">"-w"</span><span class="p">,</span><span class="w"> </span><span class="s2">"3"</span><span class="p">,</span><span class="w"> </span><span class="s2">"-b"</span><span class="p">,</span><span class="w"> </span><span class="s2">":8000"</span><span class="p">,</span><span class="w"> </span><span class="s2">"hello_django.wsgi:application"</span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"environment"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_DB_NAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_db_name}"</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_USERNAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_username}"</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_PASSWORD"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_password}"</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_HOSTNAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_hostname}"</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_PORT"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"5432"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"logConfiguration"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">      </span><span class="nt">"logDriver"</span><span class="p">:</span><span class="w"> </span><span class="s2">"awslogs"</span><span class="p">,</span><span class="w"></span>
<span class="w">      </span><span class="nt">"options"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-group"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/ecs/django-app"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-region"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${region}"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-stream-prefix"</span><span class="p">:</span><span class="w"> </span><span class="s2">"django-app-log-stream"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"nginx"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"image"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${docker_image_url_nginx}"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"essential"</span><span class="p">:</span><span class="w"> </span><span class="kc">true</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"cpu"</span><span class="p">:</span><span class="w"> </span><span class="mi">10</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"memory"</span><span class="p">:</span><span class="w"> </span><span class="mi">128</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"links"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"django-app"</span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"portMappings"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"containerPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">80</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"hostPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">0</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"protocol"</span><span class="p">:</span><span class="w"> </span><span class="s2">"tcp"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"logConfiguration"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">      </span><span class="nt">"logDriver"</span><span class="p">:</span><span class="w"> </span><span class="s2">"awslogs"</span><span class="p">,</span><span class="w"></span>
<span class="w">      </span><span class="nt">"options"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-group"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/ecs/nginx"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-region"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${region}"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-stream-prefix"</span><span class="p">:</span><span class="w"> </span><span class="s2">"nginx-log-stream"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>
<span class="w">  </span><span class="p">}</span><span class="w"></span>
<span class="p">]</span><span class="w"></span>
</code></pre>
<p>Pass the variable to the template in <em>08_ecs.tf</em>:</p>
<pre><span></span><code>data "template_file" "app" {
template = file("templates/django_app.json.tpl")

vars = {
docker_image_url_django = var.docker_image_url_django
docker_image_url_nginx  = var.docker_image_url_nginx
region                  = var.region
rds_db_name             = var.rds_db_name
rds_username            = var.rds_username
rds_password            = var.rds_password
rds_hostname            = aws_db_instance.production.address
}
}
</code></pre>
<p>Add the new logs to <em>06_logs.tf</em>:</p>
<pre><span></span><code>resource "aws_cloudwatch_log_group" "nginx-log-group" {
name              = "/ecs/nginx"
retention_in_days = var.log_retention_in_days
}

resource "aws_cloudwatch_log_stream" "nginx-log-stream" {
name           = "nginx-log-stream"
log_group_name = aws_cloudwatch_log_group.nginx-log-group.name
}
</code></pre>
<p>Update the Service so it points to the <code>nginx</code> container instead of <code>django-app</code>:</p>
<pre><span></span><code>resource "aws_ecs_service" "production" {
name            = "${var.ecs_cluster_name}-service"
cluster         = aws_ecs_cluster.production.id
task_definition = aws_ecs_task_definition.app.arn
iam_role        = aws_iam_role.ecs-service-role.arn
desired_count   = var.app_count
depends_on      = [aws_alb_listener.ecs-alb-http-listener, aws_iam_role_policy.ecs-service-role-policy]

load_balancer {
target_group_arn = aws_alb_target_group.default-target-group.arn
container_name   = "nginx"
container_port   = 80
}
}
</code></pre>
<p>Apply the changes:</p>
<pre><span></span><code>$ terraform apply
</code></pre>
<p>Make sure the app can still be accessed from the browser.</p>
<p>Now that we're dealing with two containers, let's update the deploy function to handle multiple container definitions in <em>update-ecs.py</em>:</p>
<pre><span></span><code><span class="nd">@click</span><span class="o">.</span><span class="n">command</span><span class="p">()</span>
<span class="nd">@click</span><span class="o">.</span><span class="n">option</span><span class="p">(</span><span class="s2">"--cluster"</span><span class="p">,</span> <span class="n">help</span><span class="o">=</span><span class="s2">"Name of the ECS cluster"</span><span class="p">,</span> <span class="n">required</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
<span class="nd">@click</span><span class="o">.</span><span class="n">option</span><span class="p">(</span><span class="s2">"--service"</span><span class="p">,</span> <span class="n">help</span><span class="o">=</span><span class="s2">"Name of the ECS service"</span><span class="p">,</span> <span class="n">required</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
<span class="k">def</span> <span class="nf">deploy</span><span class="p">(</span><span class="n">cluster</span><span class="p">,</span> <span class="n">service</span><span class="p">):</span>
<span class="n">client</span> <span class="o">=</span> <span class="n">boto3</span><span class="o">.</span><span class="n">client</span><span class="p">(</span><span class="s2">"ecs"</span><span class="p">)</span>

<span class="n">container_definitions</span> <span class="o">=</span> <span class="p">[]</span>
<span class="n">response</span> <span class="o">=</span> <span class="n">get_current_task_definition</span><span class="p">(</span><span class="n">client</span><span class="p">,</span> <span class="n">cluster</span><span class="p">,</span> <span class="n">service</span><span class="p">)</span>
<span class="k">for</span> <span class="n">container_definition</span> <span class="ow">in</span> <span class="n">response</span><span class="p">[</span><span class="s2">"taskDefinition"</span><span class="p">][</span><span class="s2">"containerDefinitions"</span><span class="p">]:</span>
<span class="n">new_def</span> <span class="o">=</span> <span class="n">container_definition</span><span class="o">.</span><span class="n">copy</span><span class="p">()</span>
<span class="n">container_definitions</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">new_def</span><span class="p">)</span>

<span class="n">response</span> <span class="o">=</span> <span class="n">client</span><span class="o">.</span><span class="n">register_task_definition</span><span class="p">(</span>
<span class="n">family</span><span class="o">=</span><span class="n">response</span><span class="p">[</span><span class="s2">"taskDefinition"</span><span class="p">][</span><span class="s2">"family"</span><span class="p">],</span>
<span class="n">volumes</span><span class="o">=</span><span class="n">response</span><span class="p">[</span><span class="s2">"taskDefinition"</span><span class="p">][</span><span class="s2">"volumes"</span><span class="p">],</span>
<span class="n">containerDefinitions</span><span class="o">=</span><span class="n">container_definitions</span><span class="p">,</span>
<span class="p">)</span>
<span class="n">new_task_arn</span> <span class="o">=</span> <span class="n">response</span><span class="p">[</span><span class="s2">"taskDefinition"</span><span class="p">][</span><span class="s2">"taskDefinitionArn"</span><span class="p">]</span>

<span class="n">response</span> <span class="o">=</span> <span class="n">client</span><span class="o">.</span><span class="n">update_service</span><span class="p">(</span>
<span class="n">cluster</span><span class="o">=</span><span class="n">cluster</span><span class="p">,</span> <span class="n">service</span><span class="o">=</span><span class="n">service</span><span class="p">,</span> <span class="n">taskDefinition</span><span class="o">=</span><span class="n">new_task_arn</span><span class="p">,</span>
<span class="p">)</span>
</code></pre>
<h2 id="static-files">Static Files</h2>
<p>Set the <code>STATIC_ROOT</code> in your <em>settings.py</em> file:</p>
<pre><span></span><code><span class="n">STATIC_URL</span> <span class="o">=</span> <span class="s1">'/staticfiles/'</span>
<span class="n">STATIC_ROOT</span> <span class="o">=</span> <span class="n">os</span><span class="o">.</span><span class="n">path</span><span class="o">.</span><span class="n">join</span><span class="p">(</span><span class="n">BASE_DIR</span><span class="p">,</span> <span class="s1">'staticfiles'</span><span class="p">)</span>
</code></pre>
<p>Also, turn off debug mode:</p>
<pre><span></span><code><span class="n">DEBUG</span> <span class="o">=</span> <span class="kc">False</span>
</code></pre>
<p>Update the Dockerfile so that it runs the <code>collectstatic</code> command at the end:</p>
<pre><span></span><code><span class="c"># pull official base image</span>
<span class="k">FROM</span><span class="w"> </span><span class="s">python:3.9.0-slim-buster</span>

<span class="c"># set work directory</span>
<span class="k">WORKDIR</span><span class="w"> </span><span class="s">/usr/src/app</span>

<span class="c"># set environment variables</span>
<span class="k">ENV</span><span class="w"> </span>PYTHONDONTWRITEBYTECODE <span class="m">1</span>
<span class="k">ENV</span><span class="w"> </span>PYTHONUNBUFFERED <span class="m">1</span>

<span class="c"># install psycopg2 dependencies</span>
<span class="k">RUN</span><span class="w"> </span>apt-get update <span class="se">\</span>
<span class="o">&amp;&amp;</span> apt-get -y install gcc postgresql <span class="se">\</span>
<span class="o">&amp;&amp;</span> apt-get clean

<span class="c"># install dependencies</span>
<span class="k">RUN</span><span class="w"> </span>pip install --upgrade pip
<span class="k">COPY</span><span class="w"> </span>./requirements.txt .
<span class="k">RUN</span><span class="w"> </span>pip install -r requirements.txt

<span class="c"># copy project</span>
<span class="k">COPY</span><span class="w"> </span>. .

<span class="c"># collect static files</span>
<span class="k">RUN</span><span class="w"> </span>python manage.py collectstatic --no-input
</code></pre>
<p>Next, let's add a shared volume to the Task Definition and update the Nginx conf file.</p>
<p>Add the new location block to <em>nginx.conf</em>:</p>
<pre><span></span><code><span class="nt">upstream</span><span class="w"> </span><span class="nt">hello_django</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="err">server</span><span class="w"> </span><span class="n">django-app</span><span class="p">:</span><span class="mi">8000</span><span class="p">;</span><span class="w"></span>
<span class="p">}</span><span class="w"></span>

<span class="nt">server</span><span class="w"> </span><span class="p">{</span><span class="w"></span>

<span class="w">    </span><span class="err">listen</span><span class="w"> </span><span class="err">80</span><span class="p">;</span><span class="w"></span>

<span class="w">    </span><span class="err">location</span><span class="w"> </span><span class="err">/staticfiles/</span><span class="w"> </span><span class="err">{</span><span class="w"></span>
<span class="w">        </span><span class="err">alias</span><span class="w"> </span><span class="err">/usr/src/app/staticfiles/</span><span class="p">;</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>

<span class="w">    </span><span class="nt">location</span><span class="w"> </span><span class="o">/</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="err">proxy_pass</span><span class="w"> </span><span class="n">http</span><span class="p">:</span><span class="o">//</span><span class="n">hello_django</span><span class="p">;</span><span class="w"></span>
<span class="w">        </span><span class="err">proxy_set_header</span><span class="w"> </span><span class="err">X-Forwarded-For</span><span class="w"> </span><span class="err">$proxy_add_x_forwarded_for</span><span class="p">;</span><span class="w"></span>
<span class="w">        </span><span class="err">proxy_set_header</span><span class="w"> </span><span class="err">Host</span><span class="w"> </span><span class="err">$host</span><span class="p">;</span><span class="w"></span>
<span class="w">        </span><span class="err">proxy_redirect</span><span class="w"> </span><span class="err">off</span><span class="p">;</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>

<span class="err">}</span><span class="w"></span>
</code></pre>
<p>Add the volume to <code>aws_ecs_task_definition</code> in <em>08_ecs.tf</em>:</p>
<pre><span></span><code>resource "aws_ecs_task_definition" "app" {
family                = "django-app"
container_definitions = data.template_file.app.rendered
depends_on            = [aws_db_instance.production]

volume {
name      = "static_volume"
host_path = "/usr/src/app/staticfiles/"
}
}
</code></pre>
<p>Add the volume to the container definitions in the <em>django_app.json.tpl</em> template:</p>
<pre><span></span><code><span class="p">[</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"django-app"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"image"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${docker_image_url_django}"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"essential"</span><span class="p">:</span><span class="w"> </span><span class="kc">true</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"cpu"</span><span class="p">:</span><span class="w"> </span><span class="mi">10</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"memory"</span><span class="p">:</span><span class="w"> </span><span class="mi">512</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"links"</span><span class="p">:</span><span class="w"> </span><span class="p">[],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"portMappings"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"containerPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">8000</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"hostPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">0</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"protocol"</span><span class="p">:</span><span class="w"> </span><span class="s2">"tcp"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"command"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"gunicorn"</span><span class="p">,</span><span class="w"> </span><span class="s2">"-w"</span><span class="p">,</span><span class="w"> </span><span class="s2">"3"</span><span class="p">,</span><span class="w"> </span><span class="s2">"-b"</span><span class="p">,</span><span class="w"> </span><span class="s2">":8000"</span><span class="p">,</span><span class="w"> </span><span class="s2">"hello_django.wsgi:application"</span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"environment"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_DB_NAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_db_name}"</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_USERNAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_username}"</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_PASSWORD"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_password}"</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_HOSTNAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_hostname}"</span><span class="w"></span>
<span class="w">      </span><span class="p">},</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_PORT"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"5432"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"mountPoints"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"containerPath"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/usr/src/app/staticfiles"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"sourceVolume"</span><span class="p">:</span><span class="w"> </span><span class="s2">"static_volume"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"logConfiguration"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">      </span><span class="nt">"logDriver"</span><span class="p">:</span><span class="w"> </span><span class="s2">"awslogs"</span><span class="p">,</span><span class="w"></span>
<span class="w">      </span><span class="nt">"options"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-group"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/ecs/django-app"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-region"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${region}"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-stream-prefix"</span><span class="p">:</span><span class="w"> </span><span class="s2">"django-app-log-stream"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"nginx"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"image"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${docker_image_url_nginx}"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"essential"</span><span class="p">:</span><span class="w"> </span><span class="kc">true</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"cpu"</span><span class="p">:</span><span class="w"> </span><span class="mi">10</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"memory"</span><span class="p">:</span><span class="w"> </span><span class="mi">128</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"links"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="s2">"django-app"</span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"portMappings"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"containerPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">80</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"hostPort"</span><span class="p">:</span><span class="w"> </span><span class="mi">0</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"protocol"</span><span class="p">:</span><span class="w"> </span><span class="s2">"tcp"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"mountPoints"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">      </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"containerPath"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/usr/src/app/staticfiles"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"sourceVolume"</span><span class="p">:</span><span class="w"> </span><span class="s2">"static_volume"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">],</span><span class="w"></span>
<span class="w">    </span><span class="nt">"logConfiguration"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">      </span><span class="nt">"logDriver"</span><span class="p">:</span><span class="w"> </span><span class="s2">"awslogs"</span><span class="p">,</span><span class="w"></span>
<span class="w">      </span><span class="nt">"options"</span><span class="p">:</span><span class="w"> </span><span class="p">{</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-group"</span><span class="p">:</span><span class="w"> </span><span class="s2">"/ecs/nginx"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-region"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${region}"</span><span class="p">,</span><span class="w"></span>
<span class="w">        </span><span class="nt">"awslogs-stream-prefix"</span><span class="p">:</span><span class="w"> </span><span class="s2">"nginx-log-stream"</span><span class="w"></span>
<span class="w">      </span><span class="p">}</span><span class="w"></span>
<span class="w">    </span><span class="p">}</span><span class="w"></span>
<span class="w">  </span><span class="p">}</span><span class="w"></span>
<span class="p">]</span><span class="w"></span>
</code></pre>
<p>Now, each container will share a directory named "staticfiles".</p>
<p>Build the new images and push them up to ECR:</p>
<pre><span></span><code>$ docker build -t &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest .
$ docker push &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest

$ docker build -t &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/nginx:latest .
$ docker push &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/nginx:latest
</code></pre>
<p>Apply the changes:</p>
<pre><span></span><code>$ terraform apply
</code></pre>
<p>Static files should now load correctly.</p>
<h2 id="allowed-hosts">Allowed Hosts</h2>
<p>Finally, let's lock down our application for production:</p>
<pre><span></span><code><span class="n">ALLOWED_HOSTS</span> <span class="o">=</span> <span class="n">os</span><span class="o">.</span><span class="n">getenv</span><span class="p">(</span><span class="s1">'ALLOWED_HOSTS'</span><span class="p">,</span> <span class="s1">''</span><span class="p">)</span><span class="o">.</span><span class="n">split</span><span class="p">()</span>
</code></pre>
<p>Add the <code>ALLOWED_HOSTS</code> environment variable to the container definition:</p>
<pre><span></span><code><span class="nt">"environment"</span><span class="p">:</span><span class="w"> </span><span class="p">[</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_DB_NAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_db_name}"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_USERNAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_username}"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_PASSWORD"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_password}"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_HOSTNAME"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${rds_hostname}"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"RDS_PORT"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"5432"</span><span class="w"></span>
<span class="w">  </span><span class="p">},</span><span class="w"></span>
<span class="w">  </span><span class="p">{</span><span class="w"></span>
<span class="w">    </span><span class="nt">"name"</span><span class="p">:</span><span class="w"> </span><span class="s2">"ALLOWED_HOSTS"</span><span class="p">,</span><span class="w"></span>
<span class="w">    </span><span class="nt">"value"</span><span class="p">:</span><span class="w"> </span><span class="s2">"${allowed_hosts}"</span><span class="w"></span>
<span class="w">  </span><span class="p">}</span><span class="w"></span>
<span class="p">],</span><span class="w"></span>
</code></pre>
<p>Pass the variable to the template in <em>08_ecs.tf</em>:</p>
<pre><span></span><code>data "template_file" "app" {
template = file("templates/django_app.json.tpl")

vars = {
docker_image_url_django = var.docker_image_url_django
docker_image_url_nginx  = var.docker_image_url_nginx
region                  = var.region
rds_db_name             = var.rds_db_name
rds_username            = var.rds_username
rds_password            = var.rds_password
rds_hostname            = aws_db_instance.production.address
allowed_hosts           = var.allowed_hosts
}
}
</code></pre>
<p>Add the variable to the ECS section of the variables file, making sure to add your domain name:</p>
<pre><span></span><code>variable "allowed_hosts" {
description = "Domain name for allowed hosts"
default     = "YOUR DOMAIN NAME"
}
</code></pre>
<p>Build the new image and push it up to ECR:</p>
<pre><span></span><code>$ docker build -t &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest .
$ docker push &lt;AWS_ACCOUNT_ID&gt;.dkr.ecr.us-west-1.amazonaws.com/django-app:latest
</code></pre>
<p>Apply:</p>
<pre><span></span><code>$ terraform apply
</code></pre>
<p>Test it out one last time.</p>
<p>Bring the infrastructure down once done:</p>
<pre><span></span><code>$ terraform destroy
</code></pre>
