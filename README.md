

So you're wondering, why on earth anyone would do this? Two reasons. The
first of which is getting a Kodi install working, to automatically login
and start kodi on boot isn't trivial. Since the Kodi wiki page appeared
on how to do this, it's got easier but it's a little time consuming.

Second reason is, I'm fully expecting my kodi install or pc to go up in
smoke one day. They both get daily use and it's only a matter of time
before I install a bad addon or the hardware goes up in smoke. Do I want
to spend a couple of hours reinstalling Ubuntu and then configuring Kodi?

With this I just boot my target machine off the network, select 'install'
from the installer menu and come back later and I have a working kodi
install. Optionally, I can bare metal restore from a backup, these lines
are commented out in the configs, but they might be useful to you if a
bare metal restore is something you want to do.

Overview

This set of files describes how to go about setting up tftp and dhcp
to boot the ubuntu 16.04 installer over a network. It then configures
itself using ansible and installs all the packages it needs to end up
with a working kodi install. Optionally it can restore a backup of your
previous kodi user.

Caveats

There's always one isn't there? And this one is a big one. You need to
be able to configure your DHCP server to specify boot server options. Most
DHCP services built into routers, NAS etc, don't let you do this. I run
my DHCP service on a CentOS machine so this was easier for me than it might
be for you.

You'll also need a web server
a copy of the Ubuntu install media (iso is fine)
an installed and working dhcpd, ok so if you don't already have
this one you're going to be spending a bit of time googling and 
setting it up and wondering if it's all worth it. I'm including an
example dhcpd.conf that you can edit and setup for your home network.

That said, lets get started!

Setting up tftpd

For me this was as easy as :

   yum install tftp-server

My boot/web/dhcp server runs CentOS but I'm including a link to a Debian/Ubuntu
write up which covers this part so you don't have to convert into Debianese.

Copying the source files into your webroute

   mkdir -p /var/www/html/ubuntu/1604/kodi
   mount -o loop,ro ubuntu-16.04.2-server-amd64 /mnt
   rsync -avx /mnt/* /var/www/html/ubuntu/1604/.

Copy the following files into your webroot /var/www/html/ubuntu/1604/kodi:

   kodi.yml
   kodi.service
   kodi.cfg
   kodi.conf
   rc.local.post
   rc.local.default
   updates.yml
   kodi.pkla

Now we have the source files, we setup our boot environment thusly

   mkdir -p /var/lib/tftpboot/kodi
   rsync -avx /var/www/html/ubuntu/1604/install/netboot/* \
     /var/lib/tftpboot/kodi/.

Now you need to edit /etc/xinetd.d/tftp and change disable=yes to no.

In your dhcpd.conf add the following lines near the top:

   allow booting;
   allow bootp;
   option option-128 code 128 = string;
   option option-129 code 129 = text;
   next-server 192.168.1.1;
   filename "kodi/pxelinux.0";

I specify these last two lines within host definitions in my dhcpd.conf, mainly
because I don't want to accidentally install kodi over, say, my windows
desktop by accident. So only the host I define can boot from my tftp server.

   host kodi {
      option host-name "kodi.example.com";
      hardware ethernet aa:bb:cc:dd:ee:ff;
      fixed-address 192.168.1.10;
      next-server 192.168.1.1;
      filename "kodi/pxelinux.0";
   }

Restart xinetd (where tftp.server runs from) and dhcpd :

   systemctl restart dhcpd
   systemctl restart xinetd

Now you need to configure the boot menu in the Ubuntu installer.

   cd /var/lib/tftpboot/kodi/ubuntu-installer/amd64/boot-screens

Edit txt.cfg as per the version in this git repo.

   default kodi
   label kodi
           menu label ^Install kodi
           KERNEL ubuntu-installer/amd64/linux
           IPAPPEND 1
           APPEND initrd=ubuntu-installer/amd64/initrd.gz ksdevice=eth0 locale=en_GB.UTF-8 keyboard-configuration/layoutcode=gb interface=eth0 hostname=kodi url=http://192.168.1.1/ubuntu/kodi/kodi.cfg live-installer/net-image=http://192.168.1.1/ubuntu/1604/install/filesystem.squashfs biosdevname=0 net.ifnames=0

Some explaination about some of the options there. Change the 192.168.1.1 to
be whatever your webserver is running on. The buisdevname/net.ifnames are a
hack in order to restore the eth0 device numbering. Why would I do that, am
I stuck in my old ways? Simply, I want this setup to work on ANY machine. If
I use the new method which takes the device name from the network card in
your machine, this script would be way more complicated.

Now you need to take your target machine, the one you want to install
kodi on and get into the bios to make sure pxe booting is enabled for
your network card. Without this you can't network boot. Once you've
enabled that option, reboot, and use the hotkey which enables you to 
choose your boot device. You should see your network card listed there.
Select it, and you should see a black screen where your network card
tries to get a IP address from your DHCP server and then picks up the
boot image from your tftp server. From that point onwards you just
select 'install kodi' from the installer menu and that's it!

A couple of things about how this works and why I do things the way
I do.

I couldn't get most of late_command to work in the preseed file. For
this reason a lot of what you see in rc.local.post is designed to work
around this. The /target/etc/apt directory seemed to be untouchable
during the install phase. Likewise I couldn't get any of the netcfg
options to work despite trying varying combinations based on what 
should and shouldn't be there to disable DHCP.

rc.local is a hack to keep everything local, when the machine boots
for the first time it sets up the network interface, installs ansible
and then feeds in the kodi.yml which adds the kodi user, sets up the
config needed to auto login. 

If you're using Ubuntu or Debian check out Brian Carpio's how to which
covers the DHCP/tftp portions for Debian based stuff :

http://www.briancarpio.com/2012/04/04/system-automation-part-1/

Sources, as in, the pages I googled and what actually helped:

Scott Lowe's Ubuntu netbooting howto.
http://blog.scottlowe.org/2015/05/20/fully-automated-ubuntu-install/

Kodi autostart wiki page from the Kodi wiki.
http://kodi.wiki/view/HOW-TO:Autostart_Kodi_for_Linux

Peter Hick's kernel boot options to restore eth0 dev naming.
https://blog.poggs.com/2016/11/13/get-rid-of-ubuntus-network-card-naming/

This page had the magic to get the 'power off' options to appear.
http://www.richud.com/wiki/Ubuntu_Minimal_KODI_Install
