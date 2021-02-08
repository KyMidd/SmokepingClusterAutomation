# This doc covers how to deploy a smokeping runner host
Ref: https://www.tuxtips.net/how-to-install-smokeping-on-centos-7-linux/
Ref2: https://www.wedebugyou.com/2012/11/how-to-install-and-configure-smokeping-on-centos-6/

# Enable SELinux
cat <<\EOT  >/etc/sysconfig/selinux
SELINUX=enabled
SELINUXTYPE=targeted
EOT

# Install required packages
sudo su
yum -y groupinstall "Development tools"
yum -y install epel-release perl httpd httpd-devel mod_fcgid rrdtool perl-CGI-SpeedyCGI fping rrdtool-perl perl-Sys-Syslog perl-CPAN perl-local-lib perl-Time-HiRes telnet traceroute tcptraceroute ftp vsftp dig bind-utils bc nscd iperf3 tftp* nmap wget fping.x86_64 perl-CGI-SpeedyCGI.x86_64 perl-IO-Socket-SSL.noarch perl-LDAP.noarch perl-Net-DNS.x86_64 perl-Net-Telnet.noarch perl-Socket6.x86_64 perl-libwww-perl.noarch rrdtool.i686 perl-CGI perl-core
yum -y install mod_fcgid httpd httpd-devel rrdtool rrdtool-perl fping wget curl bind-utils gcc make perl perl-Net-Telnet perl-Net-DNS perl-LDAP perl-libwww-perl perl-RadiusPerl perl-IO-Socket-SSL perl-Socket6 perl-CGI-SpeedyCGI perl-FCGI perl-RRD-Simple perl-CGI-SpeedCGI perl-ExtUtils-MakeMaker perl-rrdtool perl-core

# Install tcpping
cd /usr/bin
wget http://www.vdberg.org/~richard/tcpping
chmod 755 tcpping

# Download proper fping, build
git clone https://github.com/schweikert/fping.git
cd fping/
./autogen.sh
./configure
make
make install
rm -rf /usr/bin/fping
cp -f /usr/local/sbin/fping /usr/bin/
# Make sure fping at least version v4.0+
/usr/bin/fping -v

# Update git to required version (2+)
yum -y remove git*
yum -y install https://packages.endpoint.com/rhel/7/os/x86_64/endpoint-repo-1.7-1.x86_64.rpm
yum -y install git
# Verify at least 2.0+
git --version

# Smokeping setup
cd
mkdir smokeping
wget http://oss.oetiker.ch/smokeping/pub/smokeping-2.7.3.tar.gz
tar xfz smokeping-2.7.3.tar.gz -C /opt/
mkdir /opt/smokeping/
cd /opt/smokeping-2.7.3
./configure --prefix=/opt/smokeping
make install
cd /opt/smokeping
mkdir data var cache
chmod -R 775 cache data var

# Set hosts file
cat <<\EOT >/etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

#SmokePing
1.2.3.4  master-hostname #Master
1.2.3.4  runner1
1.2.3.4  runner2
EOT

# Update hostname file with lower-cased name to match hosts file
cat <<\EOT >/etc/hostname
runner1
EOT
hostname runner1

# Stage runner config
cat <<\EOT  >/opt/smokeping/etc/config
mode=slave
MASTER_URL=http://master-host.domain.com/smokeping
SHARED_SECRET=/opt/smokeping/etc/slave_secret
SLAVE_NAME=runner1
EOT

# Create runner secret
cat <<\EOT >/opt/smokeping/etc/slave_secret
p@ssword!
EOT
chmod 400 /opt/smokeping/etc/slave_secret

# Create smokeping service in /etc/init.d/smokeping
cat <<\EOT  >/etc/init.d/smokeping
#!/bin/sh
#set -x
#
# smokeping    This starts and stops the smokeping daemon
# chkconfig: 345 98 11
# description: Start/Stop the smokeping daemon
# processname: smokeping
# Source function library.. /etc/rc.d/init.d/functions
#. /etc/sysconfig/network
. /etc/init.d/functions
if [ "$NETWORKING" = "no" ]
then
        exit 0
fi
SLAVE_OPT="--master-url=http://master.domain.com/smokeping/smokeping.fcgi --cache-dir=/var/smokeping/ --shared-secret=/opt/smokeping/etc/slave_secret --logfile=/var/log/smokeping.log"
RETVAL=0
prog='/opt/smokeping/bin/smokeping'
lockfile=${LOCKFILE-/var/lock/subsys/smokeping}
start() {
        echo -n $"Starting $prog: "
        daemon $prog $SLAVE_OPT
        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && touch ${lockfile}
}
stop() {
        echo -n $"Stopping $prog: "
        killproc $prog
        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && rm -f ${lockfile}
}
case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  status)
        status $prog
        ;;
  restart|reload)
        stop
        start
        ;;
  condrestart)
        [ -f ${lockfile} ] && restart || :
        ;;
  *)
        echo $"Usage: $0 {start|stop|status|restart|reload|condrestart}"
        exit 1
esac
exit $?
EOT
systemctl daemon-reload
mkdir /var/smokeping
chmod +x /etc/init.d/smokeping
chmod 755 /etc/init.d/smokeping

# Restart comp
shutdown -r now

# Stage speedtest binaries
cd
wget https://github.com/mad-ady/smokeping-speedtest/archive/master.zip
unzip master.zip
mkdir /opt/smokeping/lib/Smokeping/probes/
cp -R smokeping-speedtest-master/speedtest.pm /opt/smokeping/lib/Smokeping/probes/

wget -O speedtest-cli https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py
chmod +x speedtest-cli
cp speedtest-cli /usr/bin

# Start smokeping service and set to start automatically on reboot
service smokeping start
chkconfig --level 23456 smokeping on

# Deploy Smokeping master's SSH key to permit centralized restarts
cat <<\EOF  >>/root/.ssh/authorized_keys
ssh-rsa (master key string)
EOF

# Disable IPv6 for good measure
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.default.disable_ipv6=1
