# This doc covers how to deploy a smokeping runner host
Ref: https://www.tuxtips.net/how-to-install-smokeping-on-centos-7-linux/
Ref2: https://www.wedebugyou.com/2012/11/how-to-install-and-configure-smokeping-on-centos-6/

# Disable SELinux
cat <<\EOT  >/etc/sysconfig/selinux
SELINUX=permissive
SELINUXTYPE=targeted
EOT

# Install required packages
sudo su
yum -y groupinstall "Development tools"
yum -y install epel-release perl httpd httpd-devel mod_fcgid rrdtool perl-CGI-SpeedyCGI fping rrdtool-perl perl-Sys-Syslog perl-CPAN perl-local-lib perl-Time-HiRes telnet traceroute tcptraceroute ftp vsftp dig bind-utils bc nscd iperf3 tftp* nmap wget fping.x86_64 perl-CGI-SpeedyCGI.x86_64 perl-IO-Socket-SSL.noarch perl-LDAP.noarch perl-Net-DNS.x86_64 perl-Net-Telnet.noarch perl-Socket6.x86_64 perl-libwww-perl.noarch rrdtool.i686 perl-CGI perl-core
yum -y install mod_fcgid httpd httpd-devel rrdtool rrdtool-perl fping wget curl bind-utils gcc make perl perl-Net-Telnet perl-Net-DNS perl-LDAP perl-libwww-perl perl-RadiusPerl perl-IO-Socket-SSL perl-Socket6 perl-CGI-SpeedyCGI perl-FCGI perl-RRD-Simple perl-CGI-SpeedCGI perl-ExtUtils-MakeMaker perl-rrdtool perl-core tcpdump

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

# Stage smokemail
cp /opt/smokeping-2.7.3/etc/smokemail.dist /opt/smokeping/etc/
mv /opt/smokeping/etc/smokemail.dist /opt/smokeping/etc/smokemail

# Stage config files
cd /opt/smokeping/etc
for foo in *.dist; do mv $foo `basename $foo .dist`; done

# Stage speedtest binaries
cd
wget https://github.com/mad-ady/smokeping-speedtest/archive/master.zip
unzip master.zip
mkdir /opt/smokeping/lib/Smokeping/probes/
cp -R smokeping-speedtest-master/speedtest.pm /opt/smokeping/lib/Smokeping/probes/

wget -O speedtest-cli https://raw.githubusercontent.com/sivel/speedtest-cli/master/speedtest.py
chmod +x speedtest-cli
cp speedtest-cli /usr/bin

# Stage config, can place example config like this:
cd /opt/smokeping/etc/
mv config.dist config

# Set permissions for apache
cd /opt/smokeping
chmod 600 etc/smokeping_secrets.dist
ln -s /opt/smokeping/cache /opt/smokeping/htdocs/cache
chown -R apache cache
chown -R apache data
chmod -R 775 cache data

# Setup apache, create smokeping.conf entry
cat <<\EOF  >/etc/httpd/conf.d/smokeping.conf
ScriptAlias /smokeping.cgi /opt/smokeping/htdocs/smokeping.fcgi.dist
Alias /smokeping /opt/smokeping/htdocs
FcgidMaxRequestLen 10000000
<Directory "/opt/smokeping/htdocs">
Options Indexes FollowSymLinks MultiViews ExecCGI
#Options FollowSymLinks
Require all granted
DirectoryIndex smokeping.fcgi
</Directory>
EOF

# Disable welcome page, redirect to smokeping application
cat <<\EOF  >/etc/httpd/conf.d/welcome.conf
RedirectMatch ^/$ /smokeping/smokeping.fcgi
EOF

# Stage public cert
cat <<\EOF  >/etc/pki/tls/certs/cert.crt
(public cert)
EOF

# Stage private key file
cat <<\EOF  >/etc/pki/tls/private/private.key
(private key)
EOF

# Write https config to httpd
cat <<\EOF  >/etc/httpd/conf.d/ssl.conf
Listen 443 https
SSLPassPhraseDialog exec:/usr/libexec/httpd-ssl-pass-dialog
SSLSessionCache         shmcb:/run/httpd/sslcache(512000)
SSLSessionCacheTimeout  300
SSLRandomSeed startup file:/dev/urandom  256
SSLRandomSeed connect builtin
SSLCryptoDevice builtin

<VirtualHost _default_:443>
	ErrorLog logs/ssl_error_log
	TransferLog logs/ssl_access_log
	LogLevel warn
	SSLEngine on
	SSLProtocol all -SSLv2 -SSLv3
	SSLCipherSuite HIGH:3DES:!aNULL:!MD5:!SEED:!IDEA
	SSLCertificateFile /etc/pki/tls/certs/cert.crt
	SSLCertificateKeyFile /etc/pki/tls/private/private.key
	<Files ~ "\.(cgi|shtml|phtml|php3?)$">
    	SSLOptions +StdEnvVars
	</Files>
	<Directory "/var/www/cgi-bin">
	    SSLOptions +StdEnvVars
	</Directory>
	BrowserMatch "MSIE [2-5]" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0 CustomLog logs/ssl_request_log "%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"
</VirtualHost>                                  
EOF

# Start httpd service and set to start automatically on reboot
systemctl start httpd
chkconfig httpd on

# Create smokeping service in /etc/init.d/smokeping
cat <<\EOT  >/etc/init.d/smokeping
#!/bin/sh
#set -x
#
# smokeping    This starts and stops the smokeping daemon
# chkconfig: 345 98 11
# description: Start/Stop the smokeping daemon
# processname: smokeping
# Source function library.
#. /etc/rc.d/init.d/functions
. /etc/init.d/functions

SMOKEPING=/opt/smokeping/bin/smokeping
LOCKF=/var/lock/subsys/smokeping
CONFIG=/opt/smokeping/etc/config
smokeping_options="--logfile=/var/log/smokeping.log"

[ -f $SMOKEPING ] || exit 0
[ -f $CONFIG ] || exit 0

RETVAL=0

case "$1" in
  start)
        #/opt/smokeping/bin/smokeping-service-notification.sh start
        chown -R root:apache /opt/smokeping/cache
        chmod -R 775 /opt/smokeping/cache
        chown -R root:apache /opt/smokeping/data
        chmod -R 775 /opt/smokeping/data
        echo -n $"Starting SMOKEPING: "
        daemon $SMOKEPING $smokeping_options
        RETVAL=$?
        service httpd restart
        /opt/smokeping/bin/start-slaves
        echo
        [ $RETVAL -eq 0 ] && touch $LOCKF
        ;;
  stop)
        #/opt/smokeping/bin/smokeping-service-notification.sh stop
        chown -R root:apache /opt/smokeping/cache
        chmod -R g+rwx /opt/smokeping/cache
        chown -R root:apache /opt/smokeping/data
        chmod -R g+rwx /opt/smokeping/data
        echo -n $"Stopping SMOKEPING: "
        killproc $SMOKEPING
        RETVAL=$?
        /opt/smokeping/bin/stop-slaves
        echo
        [ $RETVAL -eq 0 ] && rm -f $LOCKF
        ;;
  status)
        status smokeping
        RETVAL=$?
        ;;
  reload)
        chown -R root:apache /opt/smokeping/cache
        chmod -R g+rwx /opt/smokeping/cache
        chown -R root:apache /opt/smokeping/data
        chmod -R g+rwx /opt/smokeping/data
        echo -n $"Reloading SMOKEPING: "
        killproc $SMOKEPING -HUP
        RETVAL=$?
        echo
        ;;
  restart)
        #/opt/smokeping/bin/smokeping-service-notification.sh restart
        chown -R root:apache /opt/smokeping/cache
        chmod -R g+rwx /opt/smokeping/cache
        chown -R root:apache /opt/smokeping/data
        chmod -R g+rwx /opt/smokeping/data
        $0 stop
        sleep 3
        $0 start
        RETVAL=$?
        ;;
  condrestart)
        if [ -f $LOCKF ]; then
                $0 stop
                sleep 3
                $0 start
                RETVAL=$?
        fi
        ;;
  *)
        echo $"Usage: $0 {start|stop|status|restart|reload|condrestart}"
        exit 1
esac

exit $RETVAL
EOT
systemctl daemon-reload
mkdir /var/smokeping
chmod +x /etc/init.d/smokeping
chmod 755 /etc/init.d/smokeping

# Start smokeping service and set to start automatically on reboot
service smokeping start
chkconfig --level 23456 smokeping on

# Create smokeping secrets
cat <<\EOT >/opt/smokeping/etc/smokeping_secrets
host1:p@ssword!
host2:p@ssword2!
EOT

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
master-hostname
EOT
hostname master-hostname
