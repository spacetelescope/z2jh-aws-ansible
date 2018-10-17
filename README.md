# Zero to Jupyterhub for AWS in ansible

Ansible plays intended to set up a Jupyterhub instance from scratch. z2jh.yml tracks very closely with the AWS [zero-to-jupyterhub readthedocs](https://zero-to-jupyterhub.readthedocs.io/en/stable/) and idempotently sets up a Jupyterhub cluster. teardown.yml undoes up to a given level of the total installation, governed by which tags you specify. The default will only remove the Jupyterhub release.

Preconditions
----
* IAM role with attached policies: AmazonEC2FullAccess, IAMFullAccess, AmazonS3FullAccess, AmazonVPCFullAccess, AmazonElasticFileSystemFullAccess
* EC2 instance to serve as CI node provisioned (named [namespace]-ci) with key pair and above IAM role
* hosts file - put your CI node Public DNS (IPv4) as the only line of this
* group_vars/all
    * namespace - many things are named based on this for consistency
    * aws_region
    * ansible_ssh_private_key_file - absolute path of key file (.pem) which you use to ssh into the CI node
* Ansible installed on local machine


Zero To Jupyterhub play
----
`ansible-playbook  -i hosts z2jh.yml -v`

This will provision the AWS fixtures (EFS, S3) you need to create the infrastructure upon which Jupyterhub will run. It will create a Kubernetes cluster with kops as well and install Helm, Tiller, and download a given Jupyterhub chart and install it. Finally it will print the proxy URL where you navigate a browser to use your Jupyterhub.

It is intended to be fully idempotent so feel free to run this and it will only create the fixtures and perform the operations if necessary. For example, if you already have an EFS called [namespace]-efs, it will not create a new one, it will use that. You could run it after manually deleting your Jupyterhub release and it would simply re-install a Jupyterhub release.

Modify the config templates as needed, these will generate the configs used in the helm install.

Teardown play (colloquially named jupyterhub-to-zero)
----
`ansible-playbook -i hosts teardown.yml -v [-t TAG1 TAG2]`

This will tear down everything needed for Jupyterhub, but only what you specify. It is intended to work with a cluster that was set up with z2jh.yml, it keys off of the names that are generated therein.
I've included these tags to tear down UP THROUGH a given level of the installation.
* Default - remove Jupyterhub release
* kubernetes - remove k8s namespace and tear down kubernetes. (implies default)
    * removing kubernetes without removing the EFS means it will go as far as it can, but eventually fail.
* all-fixtures - terminate efs, s3, and the ec2 CI node. (implies kubernetes and default)
To be clear, specifying no tag will do default teardown, specifying kubernetes tag will do default and kubernetes teardown.

You may also specify specific tags for individual fixtures like fixture-ci to be sure the CI node is torn down in addition to the play being run. These do NOT imply any other work so use only in conjunction with the above tags.
* fixture-ci
* fixture-efs
* fixture-s3
