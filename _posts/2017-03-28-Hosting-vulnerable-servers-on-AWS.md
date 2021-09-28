---
title: Hosting vulnerable servers on AWS
description: Notebook on how I configured an AWS VPC with an OpenVPN server for hosting a set of vulnerable virtual images,
    intended for students to do vulnerability assessments with.
tags: security pedagogy
category: professional
---

<div class='alert alert-info'><strong>Update 11/7/2019</strong> see <a href='{% post_url 2019-11-07-gcp-vulnerable-servers %}'>this newer post</a> for an automation of the tedium of the steps in this post, albeit deployed on GCP instead of AWS.</div>

I'll be starting a new job at CU Boulder soon. One of my responsibilities will be to teach an information security management class to business school students. I don't know whether there will be a lab available where I can host VMs for students to do vulnerability assessments on, and I can't just distribute OVAs for this because of how easy it would be to boot into root. So I want to host vulnerable servers on something like AWS. Obvious problem is that vulnerable servers are... vulnerable. I don't want the boxes to be pwned before the students can start playing with them. So for the last day or so I learned about taking vulnerable VMs and launching them into a VPN on AWS. This post documents what I learned.

Went like this:

* create a VPC on AWS
* configure OpenVPN community on an Ubuntu AMI
* make a vulnerable virtual image
* Export the appliance and convert it to an AMI
* launch 1-to-many instances of your vulnerable AMIs into the VPN



### Create a VPC

I created a new VPC intended just for vulnerable servers + the OpenVPN server. I created two subnets: one default public, where I put the openVPN server, and another default private, where all the violator instances were launched.

In my config:

* vpc with subnet `172.32.0.0/16`. Your CIDR just has to be different from whatever VPCs you already have.
* created a new internet gateway, attached it to the new VPC
* Two subnets, named one 'vulnerable-public' and the other 'vulnerable-private', CIDR's `172.32.0.0/20` and `172.32.16.0/20`
* Edited the route table for the public subnet to include the new internet gateway: Destination `0.0.0.0/0` Target `<the gateway>`. This will allow OpenVPN to access the interwebs.
* While in the VPC Dashboard, check your Network ACLs, and make sure that your inbound rules allow all traffic. Also check your default security group for your new VPC, and change the source to `0.0.0.0/0`. (I don't use the default for the openvpn server, just for the instances launched into the private-only subnet).

Now I'll admit something: that last step took me late into the night. I must have kept looking over the Source error in the security rule, I could not figure out why I couldn't ssh into my openvpn server. I'm pretty certain the security group stuff here is the same as it is on the ec2 dashboard page. Note to self though, if no connectivity, it's almost certaintly security group stuff somewhere.

Note 9/26/2017: Looks like the default nowadays is for the ACLs to allow all inbound/outbound traffic by default.

It's important to note that your vulnerable VMs will not have access to the internet. For that, you would need a NAT Gateway. VPC has a handy wizard which will set that up for you, but I don't need my vulnerable VMs to have internet access.



### Configure OpenVPN community on an Ubuntu AMI

Now from the EC2 dashboard. I created a new micro instance from an official ubuntu 16.04 AWS AMI.

During the launch config:

* Change Network to the new VPC
* Change Subnet to your public one
* Click through to configure the security group. I created one that matches the one created by the official (but paid) OpenVPN Access Server marketplace AMI. That security group does this:
    * `Custom UPD | 1194 | 0.0.0.0/0`
    * `Custom TCP | 943 | 0.0.0.0/0`
    * `HTTPS | 0.0.0.0` (not necessary for the community version though)
    * `SSH | 0.0.0.0/0`
* Create a new keypair, download the pem, and if on Windows, use puttygen to load it and save the private .ppk file.

Aaaand launch and then ssh in, following the `connect` instructions.

For configuring OpenVPN server, I leaned mostly on [this blog post](https://rbgeek.wordpress.com/2012/12/13/openvpn-server-on-ubuntu-12-04-behind-nat/), and partly on [this blog post](http://agiletesting.blogspot.com/2015/01/setting-up-openvpn-server-inside-aws-vpc.html). Command history for posterity:

Assume `sudo` when not root.

Note: if you get an error about 'unable to resolve host' when you try sudo, then add `hostname` to `/etc/hosts` localhost entry.

```
apt-get update -y && apt-get upgrade -y
apt-get install openvpn easy-rsa
cd /etc/openvpn/
mkdir easy-rsa
cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
vim vars # edit parameters at bottom to whatever
su -
cd /etc/openvpn/easy-rsa
. ./vars
./clean-all
./build-ca
./build-key-server server # I didn't set a password
./build-key client # I didn't set a password.
./build-dh
cd keys
cp ca.crt server.crt server.key client.key client.crt dh2048.pem /etc/openvpn
exit # to leave root
openvpn --genkey --secret secret.key
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
gzip -d /etc/openvpn/server.conf.gz

vim server.conf  
```
Changes:
* push the CIDR for your entire VPC. In my case, `push "route 172.32.0.0 255.255.0.0"`
* change tls-auth line to be `tls-auth secret.key 0`
* uncomment `duplicate-cn` (Important!). This allows multiple clients to connect using the same certificate, which is our situation
* uncomment `client-to-client` to let metasploit exploits connect back to listeners on kali

`vim /etc/sysctl.conf`, uncomment `net.ipv4.ip_forward=1`

```
sysctl -p

iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```

* `-s` here should be the CIDR that you set in the `server` directive in server.conf, _not_ your VPC CIDR.
* also put that `iptables` line into in `/etc/rc.local`

Now test that you can launch the server.

```
openvpn --config /etc/openvpn/server.conf # to test that everything is working

openvpn --daemon --config /etc/openvpn/server.conf # when ready
```

* note that if you restart your instance, openvpn will launch as a service (e.g., `service openvpn start`, so you won't be able to run the last two commands unless you shut it down via `service openvpn stop`

<hr/>

Now, get your `client.conf` prepared for distribution to the kali boxes.

Whatever you name your client conf, don't put it in `/etc/openvpn` with a `*.conf` extension. If you do, `service openvpn start` will run it, as well as your
`server.conf` in that directory. That will make your box both a server and a client to itself, which leads to bad times and headaches.

`cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn/client.conf.for-kali`

Then, `vim client.conf.for-kali`

Changes:

* Set your open vpn public server ip in `remote ... 1194`. Add elastic ip to your openvpn aws instance when ready so that you don't have to keep changing this every time you start/stop your vpn instance.
* Add your secret.key contents inside a `<tls-auth>` block, and also add `key-direction 1` after. See my redacted client.conf below.
    See [here](http://serverfault.com/questions/483941/generate-an-openvpn-profile-for-client-user-to-import) for a discussion of inline certs.

Do _not_ put any `route` statements in `client.conf`. Let the server do the pushing.

`scp` or whatever your client.conf down off the server, and try connecting while your openvpn server is running. On Kali, just run `openvpn client.conf`. Then to test that it's all working, run `ifconfig` and look for a `tun` interface. Try pinging the openvpn host, in my case, 10.8.0.1.

Also run `route` on Kali, and confirm that you have a `destination` entry for your vpn CIDR -- in my case, `10.8.0.0 10.8.0.x 255.255.255.0`, where `x` is the ip of the destination gateway for your `tun` interface (viewable in `ifconfig`).

On Windows, with the OpenVPN community GUI, you can just rename your `client.conf` to `client.ovpn` and import that into the tool, then connect.


#### Full conf files

[Here's my redacted server.conf and client.conf](https://gist.github.com/deargle/ce70b597645dc7c7c9eaec40875faaf5)


### Make a vulnerable virtual image, then export it

I skipped this step, but here you'd create a VM however you like, making it nice and vulnerable. Then you export your image into a format that AWS likes. There are [several acceptable formats for AWS mentioned in the docs](http://docs.aws.amazon.com/vm-import/latest/userguide/export-vm-image.html), OVA included. In VirtualBox, you can export a VM as an OVA using `File > Export Appliance...`.

The important thing to note here is that only certain OSes are compatible with AWS's AMI format. [Thankfully, the list is long.](http://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html)


#### Letting the vulnerable VMs talk back to connected VPN clients

If you want to be able to run metasploit shell exploits, the vulnerable VMs need to have a route to the connected VPN clients (to the attack machines). This is because metasploit launches
listeners, then executes payloads on victim machines that reach out to the listener. The solution I came up with for doing this is to put each of the vulnerable VMs on the VPN, too, but to
change their client.conf a bit.


* Install openvpn to your image.
* Change client.conf `remote` directive to point to the _private ip address_ of your openvpn instance, not the public one (remember, the instances in the private network don't have internet connectivity)
* Add two more lines to `client.conf` that goes on the vulnerable vms (do _not_ put these in the client.conf that Kali uses!):
	* `route-nopull` (this blocks the vpn connection from clobbering the private network route that already exists for your VPC via the 'push route' statements, in my case, 172.32.0.0/16.
	* `route 10.8.0.0 255.255.255.0` (Set your vpn server's `server` directive CIDR network here. This lets the vulnerable servers talk to clients on the VPN network, when combined with the `client-to-client` directive in server.conf)
        * This route statement is necessary because we said `route-nopull`, which otherwise would have set the `10.8.0.0` for us.

Edit `/etc/defaults/openvpn` on the vulnerable vms and add a line for `AUTOSTART="client"`, then put your edited client.conf in /etc/openvpn. With it there, it should be automatically started on boot.

If you forgot to do this step before you imported your VM image into AWS, you can edit one of your launched images, then make a new AMI image based on that running instance. Faster than reuploading the whole thing to S3.



### Convert your image to an AMI

You'll need the [AWS CLI](https://aws.amazon.com/cli/). Download it, and then give it your root credentials by running `aws configure` so that you can follow the next steps.

The process for creating an AMI involves:

* uploading your `.ova` to an S3 bucket,
* creating an IAM `VM Import Service Role`,
* invoking the magic spells in the aws console to import the image from S3 into an AMI.

Note: upload your `.ova` into the S3 bucket in the region where you want your `openvpn` server to run. When you run the commands to convert the `.ova` into an AMI, the bucket region will determine in which region the AMI ends up.
If you import it into the wrong region, you can copy your AMI to a different region [thusly](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html#copying-an-ami)

Docs [here](http://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html). First follow [these steps for creating a VM Import Service Role](http://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html#import-vm-image). Then, upload the VM image file to S3 (I used the browser interface to upload). Then follow the last step to initiate the import transfer.

Went smoothly for me, although the actual import (the last step) took 10+ minutes. When you create your own AMI, AWS launches a copy of your instance appliance and then takes a snapshot of it. That snapshot will be associated with your new AMI, and is not deletable unless you first unregister your AMI. So be aware, you'll be paying for the storage of that snapshot as long as you have the AMI. Note that your AMI will
only show up in the view for the region where you deployed it.

Note: When you convert your `.ova` into an AMI, any network interface that it used to have are replaced with one that will work with EC2. The new network interface should
automatically obtain a private ip in the subnet that it's launched in when it starts up.

Note 9/26/2017: When I used the aws cli on Linux, I had to run `sudo ntpdate time.nist.gov` before the s3 commands would work.




### Launch a gazillion Violators

Or whatever.
* Launch them into the correct `Network`, and for the `Subnet`, into the _private_ subnet so that they don't get a public ip.
* "Proceed without a keypair"
* Change to a security group setting that allows "All Traffic" inbound.
* You can launch multiple at once at "Step 3: Configure Instance Details" > "Number of instances"

Try pinging their ips from a client that's connected to the VPN. Then try exploiting them. **Important:** In `msf`, for `LHOST`, use kali's _vpn ip address_ from `tun0`,
but you can target the private-ip address of the vulnerable vms (the `172.` addresses for me -- you don't need to target or know the vulnerable vm's vpn tun0
ip address).

Fist-pump ~~three~~ six times if great success.

<hr/>

Note: Because `client-to-client` is enabled, students can theoretically attack other students' kali instances if they're both connected to the vpn at the same time.
Kali doesn't have a listening ssh server by default, so known-password isn't a vector. (They'll likely not have changed the password from what it
was when you distributed VM instances to them...)
