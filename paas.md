# Creating your own Platform as a Service

## Chef setup

Chef offers a free hosted service for up to five servers.  That's plenty for this exercise so go to the [registration page](https://www.getchef.com/account) and create yourself a user.  At some point it will prompt you to generate and save a key, do that and download it.

* [http://manage.opscode.com](http://manage.opscode.com)

Once you have signed up you can download a knife config file and generate a validation key from the [https://manage.opscode.com/organizations](Organizations) page.  We can save those down and then move them to a local working directory.

### Prepare Working Environment

Create a `~/paas` working directory and configure your local chef tools like this ( change the Download location to match the files you downloaded above ) :

```
$ mkdir -p ~/paas/.chef
$ cd ~/paas
$ mv ~/Downloads/<username>.pem .chef/
$ mv ~/Downloads/knife.rb .chef/
$ mv ~/Downloads/<username>-validator.pem .chef/

```

### Clone the Deis Repository

Clone the deis project into your paas working directory:

```
$ cd ~/paas
$ git clone https://github.com/opdemand/deis.git
```

### Install Pre-reqs

Assuming you have a working `Ruby 1.9.3+` and the `bundler` gem installed you should be able to use the `Gemfile` from the deis project to ensure you have all the necessary tools:

```
$ cd ~/paas/deis
$ bundle install
```

*I had some errors installing the eventmachine gem and had to follow [this fix](https://github.com/gitlabhq/gitlabhq/issues/1051#issuecomment-9176547) to get bundle install to work*

### Test Chef Connectivity

To make sure we configured chef correctly and installed knife as part of the bundle we can run a quick knife command:

```
$ bundle exec knife client list
<USERNAME>-validator
```

### Create an Environment for Deis

This command will open up a `vim` editor,  save it without changing.

```
EDITOR=vi knife environment create deis

```


### Upload the Deis Cookbooks

If that went well we can upload our cookbooks:

```
cd ~/paas/deis
$ bundle exec berks install
$ bundle exec berks upload
```
### Create Deis Databags

Deis uses some databags to help manage users and state.  We can create them like this:

```
$ bundle exec knife data bag create deis-users
Created data_bag[deis-users]
$ bundle exec knife data bag create deis-formations
Created data_bag[deis-formations]
$ bundle exec knife data bag create deis-apps
Created data_bag[deis-apps]
```

## Prepare Infrastructure

I'm using Rackspace cloud servers for this as I have the (http://developer.rackspace.com/blog/developer-love-welcome-to-the-rackspace-cloud-developer-discount.html)[Rackspace Developer Discount] which is enough discount to host this for free.

Since Deis will want your rackspace credentials to configure worker nodes I recomment creating a user under (https://mycloud.rackspace.com/account#users/create)[User Management] in your account to use for this.

### Create a Cloud Load Balancer

Log into mycloud.rackspace.com and click on the (https://mycloud.rackspace.com/load_balancers)[Load Balancers] button.  Select the Dallas Region (DFW) and hit `Create Load Balancer`.

* Set the Name to `deis` and check the region is set to `Dallas (DFW)` and hit `Create Load Balancer`

* Take note of the public IP of the Load Balancer, we'll need it later.

### Wildcard DNS

Deis' proxy layer requires you to set up Wildcard DNS to point to your proxy layer.  There are many ways to achieve this here are two options:

1. Rackspace Cloud DNS can host wildcard DNS entries, if you already have DNS hosted by rackspace using Cloud DNS simply add an A record for `*.deis` under your domain and point it to the IP of your load balancer.

2. The (http://xip.io)[xip.io] domain does wildcard DNS based on your IP.  We can use this with our Cloud Load Balancer to load balance our applications.   My Load Balancer has a public IP of `162.242.140.21` therefore my wildcard domain will be `162.242.140.21.xip.io`.   Remember this.

### Configure Knife for Rackspace

The bundle install above already installed the rackspace knife plugin so we just need to add some details to `.chef/knife.rb`.   

```
$ cat <<'EOF' >> $HOME/.chef/knife.rb
knife[:rackspace_api_username] = "#{ENV['OS_USERNAME']}"
knife[:rackspace_api_key]      = "#{ENV['OS_PASSWORD']}"
knife[:environment]            = 'deis'
knife[:rackspace_version]      = 'v2'
knife[:rackspace_region]       = :dfw
EOF
```

### Install Rackspace Nova Client

We also need the Nova client:

```
$ sudo pip install rackspace-novaclient
$ cat <<'EOF' >> ~/paas/.chef/openrc
export OS_AUTH_URL=https://identity.api.rackspacecloud.com/v2.0/
export OS_AUTH_SYSTEM=rackspace
export OS_REGION_NAME=DFW
export OS_USERNAME=<RACKSPACE_USERNAME>
export NOVA_RAX_AUTH=1
export OS_PASSWORD=<RACKSPACE_API_KEY>
export OS_NO_CACHE=1
export OS_TENANT_NAME=<RACKSPACE_USERNAME>
EOF
$ source ~/paas/.chef/openrc
```

### Test Rackspace Connectivity

Make sure you can connect to Rackspace with Knife:

```
$ bundle exec knife rackspace server list
Instance ID  Name  Public IP  Private IP  Flavor  Image  State
```

Make sure you can connect to Rackspace with nova:

```
$ nova list
+--------------------------------------+-----------------+--------+------------+-------------+----------------------------------------------------------------------------------------+
| ID                                   | Name            | Status | Task State | Power State | Networks                                                                               |
+--------------------------------------+-----------------+--------+------------+-------------+----------------------------------------------------------------------------------------+
```

## Build a base image for Deis

### Launce a new instance:

If we create a base image and pre-install some software we'll get a faster booting system for auto-provisioning:

```
$ bundle exec knife rackspace server create \
  --image '80fbcb55-b206-41f9-9bc2-2dd7aac6c061' \
  --node-name 'deis-base-image' \
  --run-list 'recipe[deis::default]' \
  --flavor 'performance1-1'
...
...
Instance ID: 56760bf1-b977-405e-9348-f70b15a14b87
Host ID: 97da00a12312a7e455bda70c6dfab8833953e2a03b081aeedfd68152
Name: deis-base-image
Flavor: 1 GB Performance
Image: Ubuntu 12.04 
Metadata: []
Public DNS Name: 23-253-69-98.static.cloud-ips.com
Public IP Address: 23.253.69.98
Private IP Address: 10.208.101.31
Password: **************
Environment: deis
```

Take note of the `Instance ID`, `Public IP Address` and `Password` 



```
$ DEIS_IP=<IP_OF_SERVER>
$ ssh-copy-id
$ scp contrib/rackspace/*.sh deis-ops@$DEIS_IP:~/
$ ssh deis-ops@$DEIS_IP 'sudo apt-get -yqq install inotify-tools'
$ ssh deis-ops@$DEIS_IP 'sudo ~/prepare-rackspace-image.sh'
$ ssh deis-ops@$DEIS_IP 'sudo rm -rf /etc/chef'
```

## Create an image from this server

```
$ nova image-create deis-base-image deis-base-image
/usr/lib/python2.7/dist-packages/gobject/constants.py:24: Warning: g_boxed_type_register_static: assertion 'g_type_from_name (name) == 0' failed
  import gobject._gobject

```

After a few minutes you should see this response to running `nova image-list`, if you're impatient like me wrap your command with a `watch`:

```
$ watch 'nova image-list | grep deis'
| df958d26-6515-4dd9-a449-920e74ea93a2 | deis-base-image                                              | ACTIVE | 0fc7f68b-176d-49a9-82ff-2d5893d32acd |

```

## Delete the Base Instance

No need to keep it around and keep paying for it:

```
$ bundle exec knife rackspace server delete <INSTANCE_ID> --purge
```

## Create the Deis Controller server

### Create Databags

Deis uses some databags for keeping track of users and state:

```
$ knife data bag create deis-users
$ knife data bag create deis-formations
$ knife data bag create deis-apps
```


### Launch the Server

Launch the server from the image you created earlier:

```
$ knife rackspace server create \
  --image "29c0c2cd-24e1-4b36-bcb7-519d81ea11ff" \
  --rackspace-metadata "{\"Name\": \"deis-controller\"}" \
  --rackspace-disk-config MANUAL \
  --server-name deis-controller \
  --node-name deis-controller \
  --flavor 'performance1-2'
```

Take note of the `Instance ID`, `Public IP Address` and `Password`.

Chuck an entry in your hosts file and upload your ssh key:

```
$ sudo sh -c "echo '<IP_OF_SERVER> deis' >> /etc/hosts"
```

### Modify Chef Admin Group

On the Chef management website click (https://manage.opscode.com/groups/admins/edit)[Groups] and add the `deis-controller` client and your validator client to the `admins` group.

### Converge the Deis Controller Server

Edit the `deis-controller` node via this command: 

```
$ EDITOR=vi knife node edit deis-controller
```

make it look like this:

```
{
  "name": "deis-controller",
  "chef_environment": "deis",
  "normal": {
    "tags": [

    ]
  },
  "run_list": [
    "recipe[deis::controller]"
  ]
}

```

then converge the node by running chef client on it:

```
$ ssh deis-ops@deis sudo chef-client
```

## Testing Deis

### Install the Deis Client with pip

The Deis client is written in python and can be installed by `pip`:

```
$ sudo pip install deis  
```

### Register Admin User

First user to register becomes the Admin.

```
$ deis register http://deis
username: admin
password: 
password (confirm): 
email: admin@example.com
Registered admin
Logged in as admin
$ deis keys:add ~/.ssh/id_rsa.pub 
Uploading SSH_KEY to Deis...done
```

check the web server is serving content by browsing to (http://deis)[http://deis] and entering your admin credentials.

### Teach Deis your provider credentials

Deis will automatically provision worker nodes if you teach it your credentials.

We already have our Rackspace credentials saved to `~/paas/.chef/openrc` but Deis wants them named differently:

```
$ export RACKSPACE_USERNAME=$OS_USERNAME
$ export RACKSPACE_API_KEY=$OS_PASSWORD
$ deis providers:discover
No EC2 credentials discovered.
Discovered Rackspace credentials: ****************
Import Rackspace credentials? (y/n) : y
Uploading Rackspace credentials... done
No DigitalOcean credentials discovered.
No Vagrant VMs discovered.
```

## Deploy Formations & Layers

??? TALK ABOUT THESE???

Single Host

Create formation:

```
$ deis formations:create dev --domain=162.242.140.21.xip.io
Creating formation... done, created dev
See `deis help layers:create` to begin building your formation
```

Use the wildcard domain from earlier in the `--domain` argument.

Create layers:

```
$ deis layers:create dev nodes rackspace-dfw --proxy=y --runtime=y
Creating nodes layer... done in 4s
```

Scale Nodes:

```
$ deis nodes:scale dev nodes=2
Scaling nodes... but first, coffee!
done in 345s
Use `deis create --formation=dev` to create an application
```

## Update Cloud Load Balancer

Add these two nodes to the (https://mycloud.rackspace.com/load_balancers)[Cloud Load Balancer] we created earlier.

This is simple to do through the GUI:

* Click on your load balancer and under `Nodes` click the `Add Cloud Servers` button.
* Check the box beside the two `dev-nodes` servers and click `Add Selected Servers`.


## Deploy an Application

So great, you have a PaaS, but what do you do now?  Deploy some apps of course!

### NodeJS Example App


Download the NodeJS example application so like:

```
$ mkdir -p ~/paas/apps
$ cd ~paas/apps
$ git clone https://github.com/opdemand/example-nodejs-express.git
$ cd example-nodejs-express
```

### Create an Application in Deis

Use the Deis command line tool to create a new application:

```
$ deis create --formation=dev --id=firstapp
Creating application... done, created firstapp
Git remote deis added
```

### Push your Application to Deis

```
$ git push deis master                     
Counting objects: 167, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (72/72), done.
Writing objects: 100% (167/167), 27.37 KiB | 0 bytes/s, done.
Total 167 (delta 92), reused 167 (delta 92)
-----> Node.js app detected
-----> Requested node range: 0.10.x
-----> Resolved node version: 0.10.24
-----> Downloading and installing node
-----> Installing dependencies
...
...
-----> Compiled slug size is 5.6M

       Launching... 
done, v2

-----> firstapp deployed to Deis
       http://firstapp.xip.io

       To learn more, use `deis help` or visit http://deis.io

To git@deis:firstapp.git
 * [new branch]      master -> master
```

## Did it work ?

Open your web browser to the URL in the output of the previous command.  In my case this was `http://firstapp.162.242.140.21.xip.io`.

If everything worked the text in the browser window should read `Powered by Deis`.

## Scale your application

Expecting visitors?  Let's scale this to be running five copies of the application:

```
$ deis scale web=5
Scaling containers... but first, coffee!
done in 58s

=== firstapp Containers

--- web: `node server.js`
web.1 up 2014-01-18T22:57:55.375Z (dev-nodes-1)
web.2 up 2014-01-18T23:00:56.877Z (dev-nodes-2)
web.3 up 2014-01-18T23:00:56.885Z (dev-nodes-1)
web.4 up 2014-01-18T23:00:56.893Z (dev-nodes-2)
web.5 up 2014-01-18T23:00:56.901Z (dev-nodes-1)
```
## Update your application

New content?  Let's go ahead and update our application.

Edit `servers.js` and replace `message = "deis"` with `message = "Deis and Rackspace!"`

### Ship it!

Commit and push the new code:

```
$ git commit -am 'powered by rackspace'
$ git push deis master
Counting objects: 5, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 302 bytes | 0 bytes/s, done.
Total 3 (delta 2), reused 0 (delta 0)
-----> Node.js app detected
-----> Requested node range: 0.10.x
...
...
       Launching... done, v3

-----> firstapp deployed to Deis
       http://firstapp.162.242.140.21.xip.io
```

### Test it!

Open your web browser to the URL in the output of the previous command.  In my case this was `http://firstapp.162.242.140.21.xip.io`.

If everything worked the text in the browser window should read `Powered by Deis and Rackspace!`.





