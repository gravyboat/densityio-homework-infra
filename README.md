# densityio-homework-infra
The infrastrucure that supports the requirements from
https://github.com/DensityCo/devops-homework

# What This Repo Does

This repo uses AWS with a ready to go cloudformation template (in a new VPC)
from https://aws.amazon.com/quickstart/architecture/heptio-kubernetes/ with
a few changes to set up a Kubernetes cluster with a bastion server, a load
balancer, and a postgres RDS instance. There are a few manual steps here due
to the time limit. 

# Infrastructure Setup
Set up your AWS account if you don’t have one already.

Start by configuring an SSH key using the following instructions:
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair

Head over to the cloudformation page and click on “create a new stack” then
Load the template file from this repo into Cloudformation. Hit next and
fill out the fields that don’t already have default values. This stack will
allow you to create multiple instances (Prod and QA for example) which you
could then deploy to using Kubernetes. In this example we’re only configuring
one environment, but you could simply duplicate the stack with a new name and
availability zone to create a QA environment as the whole thing is self contained.
Once the setup is complete (should take about 10 minutes), we can now deploy
our application using Kubernetes.

# Configure Connectivity for Administration Purposes

To connect via SSH go to the Cloudformation stack you just created, look at
“Outputs”, and then use the SSHProxyCommand value to connect to the
Kubernetes master. You can also set up a local kubectl configuration using
the GetKubeConfigCommand value from the “Outputs” section of the stack if
you have that configured (and you probably want to configure this for the
sake of secrets if this were a production app so install kubectl using the
instructions here: https://kubernetes.io/docs/tasks/tools/install-kubectl/,
run the GetKubeConfigCommand script, then export it via
`export KUBECONFIG=$(pwd)/kubeconfig` so it’s available. You can test that
everything worked by running kubectl get nodes and you should see the master
and 2 other systems with a role of “none”).

# Enabling RDS Access

Next we need to allow traffic through to the RDS instance from our Kubernetes
cluster. For the sake of simplicity we’re just going to open up the RDS
instance in this example. Head over to the VPC dashboard, find the security
group associated with your RDS instance (if you don’t know what it is go to
the instance within RDS and scroll down to all the details and you’ll see the
security group), then go to “inbound rules”, edit it, and add a traffic rule
for ALL Traffic, ALL protocol, ALL port range, and 0.0.0.0/0 for the source.
Obviously this isn’t the most secure and the template could be modified to
only allow requests from the Kubernetes cluster.

# Importing Our Postgres Schema

Next we need to import our initial schema for the postgres database. Connect
over to the Kubernetes master with the SSH steps above, then using the schema
file from our code repo
(https://github.com/gravyboat/densityio-homework-app/blob/master/schema.sql)
load in the initial schema:
`psql -Udensity -h address.us-east-2.rds.amazonaws.com -d MyDatabase --password < schema.sql`.
Enter the password when prompted and the initial schema will be loaded into our
RDS instance. At this point the infrastructure setup is ready to go for
production deployments.

# SSL Configuration

Finally we need to configure an SSL certificate for our Load Balancer. For
this we can simply set up an SSL on our domain, go to our Load Balancer
configuration, modify the listener, add HTTPS, then add the SSL certificate
from our domain. Requesting an SSL through AWS is the easiest way to do this
since we are already using their services. We also need to edit our security
group’s inbound rules to support HTTPS.

In the event the load balancer with SSL doesn't do the job something like
https://github.com/jetstack/cert-manager could be used.

# Next Steps

Once you've finished reading through this head over to
https://github.com/gravyboat/densityio-homework-app
for the app itself.
