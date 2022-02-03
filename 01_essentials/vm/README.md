# Creating Virtual Machine


## Activate Cloud Shell
Cloud Shell is a virtual machine that is loaded with development tools. It offers a persistent 5GB home directory and runs on the Google Cloud. Cloud Shell provides command-line access to your Google Cloud resources.

In the Cloud Console, in the top right toolbar, click the Activate Cloud Shell button.

Hit Continue

It takes a few moments to provision and connect to the environment. When you are connected, you are already authenticated, and the project is set to your PROJECT_ID.

`gcloud` is the command-line tool for Google Cloud. It comes pre-installed on Cloud Shell and supports tab-completion.

You can list the active account name with this command:

```bash
gcloud auth list

# Output

Credentialed accounts:
 - <myaccount>@<mydomain>.com (active)
```

You can list the project ID with this command:

```bash
gcloud config list project

# Output

[core]
project = <project_ID>
```

## Understanding Regions and Zones
Certain Compute Engine resources live in regions or zones. A region is a specific geographical location where you can run your resources. Each region has one or more zones. For example, the `us-central1` region denotes a region in the Central United States that has zones `us-central1-a`, `us-central1-b`, `us-central1-c`, and `us-central1-f`.

Resources that live in a zone are referred to as zonal resources. Virtual machine Instances and persistent disks live in a zone. To attach a persistent disk to a virtual machine instance, both resources must be in the same zone. Similarly, if you want to assign a static IP address to an instance, the instance must be in the same region as the static IP.

## Create a new instance from the Cloud Console

In the Cloud Console, on the **Navigation menu**, click **Compute Engine > VM Instances**.

This may take a minute to initialize for the first time.

To create a new instance, click CREATE INSTANCE.

There are many parameters you can configure when creating a new instance. Use the following:

|Field|Value|Additional Information|
|--------------|--------------|--------------|
|Name|gcelab|Name for the VM instance|
|Region|europe-west3-a (Frankfurt)|For more information about regions, see [Regions and Zones](https://cloud.google.com/compute/docs/regions-zones).|
|Zone|europe-west3-a (Frankfurt)|Note: Remember the zone that you selected: you'll need it later. For more information about zones, see [Regions and Zones](https://cloud.google.com/compute/docs/regions-zones).|
|Series|N1|Name of the series|
|Machine Type|2 vCPU|This is an (n1-standard-2), 2-CPU, 7.5GB RAM instance. Several machine types are available, ranging from micro instance types to 32-core/208GB RAM instance types. For more information, see [Machine Types](https://cloud.google.com/compute/docs/machine-types). Note: A new project has a default [resource quota](https://cloud.google.com/compute/quotas), which may limit the number of CPU cores. You can request more.|
|Boot Disk|New 10 GB balanced persistent disk OS Image: Debian GNU/Linux 10 (buster)|Several images are available, including Debian, Ubuntu, CoreOS, and premium images such as Red Hat Enterprise Linux and Windows Server. For more information, see Operating System documentation.|
|Firewall|Allow HTTP traffic|Select this option in order to access a web server that you'll install later. Note: This will automatically create a firewall rule to allow HTTP traffic on port 80.|


Click **Create**.

It should take about a minute for the machine to be created. After that, the new virtual machine is listed on the **VM Instances** page.

To use SSH to connect to the virtual machine, in the row for your machine, click **SSH**.

This launches an SSH client directly from your browser.

## Install an NGINX web server
Now you'll install an NGINX web server, one of the most popular web servers in the world, to connect your virtual machine to something.

In the SSH terminal, to get root access, run the following command:

```bash
sudo su -
```

As the root user, update your OS:

```bash
apt-get update

# Output

Get:1 http://security.debian.org stretch/updates InRelease [94.3 kB]
Ign http://deb.debian.org strech InRelease
Get:2 http://deb.debian.org strech-updates InRelease [91.0 kB]
...
```

Install NGINX:

```bash
apt-get install nginx -y

# Output
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following additional packages will be installed:
...
```

Confirm that NGINX is running:

```bash
ps auwx | grep nginx

# Output
root      2330  0.0  0.0 159532  1628 ?        Ss   14:06   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data  2331  0.0  0.0 159864  3204 ?        S    14:06   0:00 nginx: worker process
www-data  2332  0.0  0.0 159864  3204 ?        S    14:06   0:00 nginx: worker process
root      2342  0.0  0.0  12780   988 pts/0    S+   14:07   0:00 grep nginx
```

To see the web page, return to the Cloud Console and click the External IP link in the row for your machine, or add the External IP value to http://EXTERNAL_IP/ in a new browser window or tab.

This NGINX default web page should open:


## Create a new instance with gcloud