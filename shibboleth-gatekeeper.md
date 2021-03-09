# Shibboleth SP GateKeeper
In our network, we wanted to enable remote access to some internal websites, without using our internal VPN.
However, those websites were not designed to authenticate remote users. Therefore, we designed an 
[Apache httpd](https://httpd.apache.org/) reverse proxy, which authenticates all remote users
using a [Shibboleth Service Provider](https://wiki.shibboleth.net/confluence/display/SP3/Home) (SP).

The SP is connected to our institutional [Shibboleth Identity Provider](https://wiki.shibboleth.net/confluence/display/IDP4/Home) (IdP),
member of the Italian [IDEM](https://idem.garr.it/) federation and [eduGAIN](https://edugain.org/).

## Hostnames and DNS
We assume that the gatekeeper has hostname **sp.example.com**, while the IdP **idp.example.com**.

The gatekeeper needs to protect two websites:
- **sv1.example.com** - ``192.0.2.1``
- **sv2.example.com** - ``192.0.2.2``

To do so, change the DNS registration and modify those websites to ``CNAME`` records, pointing to **sp.example.com**.
The gatekeeper will then reverse-proxy the connection directly to the IP address of those websites.

## GateKeeper configuration
The *GateKeeper* is installed on a dedicated virtual machine. For this example, we used [CentOS](https://www.centos.org/) 8 Linux, but
these instructions can be easily adapted for other distributions.

This guide is mildly based on the 
[official IDEM](https://github.com/ConsortiumGARR/idem-tutorials/blob/master/idem-fedops/HOWTO-Shibboleth/Service%20Provider/CentOS/HOWTO%20Install%20and%20Configure%20a%20Shibboleth%20SP%20v3.x%20on%20CentOS%207%20(x86_64).md) guide, 
even though this SP will not be part of any federation.

### Software installation
Start by installing the [Apache httpd](https://httpd.apache.org/) server:
```bash
dnf install httpd mod_ssl
```
Then, install Shibboleth SP using the official repositories. Grab the correct repo file from https://shibboleth.net/downloads/service-provider/RPMS/,
copy it inside ``/etc/yum.repos.d`` and then install
```bash
dnf install shibboleth.x86_64
```

Finally, open the ports ``80`` and ``443`` on the firewall
```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

### Apache configuration
#### Initial configuration
At the beginning, set up a simple TLS-enabled `VirtualHost` that is pointing to **sp.example.com**. 
A good practice is to use an automatic tool to automatize the issuance and the installation of certificates, 
like [certbot](https://certbot.eff.org/lets-encrypt/centosrhel8-apache).

In this case, install certbot and then issue a certificate:
```bash
dnf install epel-release
dnf install certbot python3-certbot-apache
certbot run --apache -d sp.example.com
```
Certbot should automatically issue a certificate for **sp.example.com** and properly configure
Apache with those. Verify by opening a web browser on the page https://sp.example.com.

#### Reverse proxy configuration
Then, configure the two virtual hosts with the reverse proxy. The exact configuration
must be tuned to the specific service, so use this just as a template.

Start by issuing the certificates
```bash
certbot certonly --apache -d sv1.example.com sv2.example.com
```
Then, create a file per each service in `/etc/httpd/conf.d`. For instance,
the file `/etc/httpd/conf.d/sv1.example.com.conf` can be:
```
<VirtualHost *:443>
    SSLEngine on

    SSLCertificateFile          /etc/letsencrypt/live/sv1.example.com/fullchain.pem
    SSLCertificateKeyFile       /etc/letsencrypt/live/sv1.example.com/privkey.pem

    # enable HTTP/2, if available
    Protocols h2 http/1.1

    # HTTP Strict Transport Security (mod_headers is required) (63072000 seconds)
    Header always set Strict-Transport-Security "max-age=63072000"

    # VirtualHost settings
    ServerName sv1.example.com:443
    ServerAdmin admin.example.com
    
    # Reverse proxy
    ProxyPass "/" "http://192.0.2.1:80/"
    ProxyPassReverse "/" "http://192.0.2.1:80/"

    # Shibboleth authentication over the entire virtual host
    <Location />
      AuthType shibboleth
      ShibRequestSetting requireSession 1
      <RequireAny>
        Require shib-session
        Require ip 10.0.0.0/8
      </RequireAny>
    </Location>
</VirtualHost>
```
With this configuration, local clients, which have a private IP address
in the network `10.0.0.0/8`, are not required to authenticate. Feel free to tune this
configuration to your needs.

At the end, test the configuration and start Apache
```bash
httpd -t
systemctl --now enable httpd
```

### Shibboleth SP configuration
First, get the metadata of your IdP
```bash
curl https://idp.inrim.it/idp/shibboleth > /etc/shibboleth/idp-example-com-local-metadata.xml
```

Edit the ``/etc/shibboleth/shibboleth2.xml`` file, adding
```xml
    <ApplicationDefaults entityID="https://sp.example.com/shibboleth"
        REMOTE_USER="eppn subject-id pairwise-id persistent-id"
        cipherSuites="DEFAULT:!EXP:!LOW:!aNULL:!eNULL:!DES:!IDEA:!SEED:!RC4:!3DES:!kRSA:!SSLv2:!SSLv3:!TLSv1:!TLSv1.1">

...

            <SSO entityID="https://idp.example.com/idp/shibboleth">
              SAML2
            </SSO>
...

            <Handler type="MetadataGenerator" Location="/Metadata" signing="false">
                    <EndpointBase>https://sp.example.com/Shibboleth.sso</EndpointBase>
                    <EndpointBase>https://sv1.example.com/Shibboleth.sso</EndpointBase>
                    <EndpointBase>https://sv2.example.com/Shibboleth.sso</EndpointBase>
            </Handler>

...

        <Errors supportContact="hostmaster@example.com"
            helpLocation="/about.html"
            styleSheet="/shibboleth-sp/main.css"/>

...

        <MetadataProvider type="XML" validate="true" path="idp-example-com-local-metadata.xml"/>


```

Create SP metadata Signing and Encryption credentials (taken from IDEM GARR Guide):
```bash
cd /etc/shibboleth
./keygen.sh -u shibd -g shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-signing -f
./keygen.sh -u shibd -g shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-encrypt -f
LD_LIBRARY_PATH=/opt/shibboleth/lib64 /usr/sbin/shibd -t
```
Make sure that the command `hostname -f` returns **sp.example.com**. If not, please fix it using the
``hostnamectl`` command.

At the end, reload Apache and Shibboleth
```bash
systemctl --now enable shibd
systemctl restart httpd
```

## Identity Provider configuration
I assume that you have a working Shibboleth IdP v4. If so, on the IdP, download the SP metadata
and load them.
```bash
curl https://sp.example.com/Shibboleth.sso/Metadata > /opt/shibboleth-idp/metadata/sp-example-com-metadata.xml
```
According to https://help.it.ox.ac.uk/iam/federation/using-one-shibboleth-service-provider-multiple-virtual-hosts, this metadata
must be edited, making sure that the tags `md:SingleLogoutService` refer only to the main hostname **sp.example.com**. Therefore,
delete all other `md:SingleLogoutService` that are referring to the services **sv1.example.com** and **sv2.example.com**.

Edit the file `/opt/shibboleth-idp/conf/metadata-providers.xml`, adding this local metadata provider
```xml
...
   <MetadataProvider id="SPExampleLocalMetadata"  xsi:type="FilesystemMetadataProvider" metadataFile="%{idp.home}/metadata/sp-example-com-metadata.xml"/>
...
```

Then, reload the IdP:
```bash
systemctl restart jetty
```

## Test
Test the configuration by visiting https://sv1.example.com from different locations. After successful authentication with the IdP, then
the website should be visible. This authentication is completely transparent for the website.

### References
- https://help.it.ox.ac.uk/iam/federation/using-one-shibboleth-service-provider-multiple-virtual-hosts
- https://github.com/ConsortiumGARR/idem-tutorials

#### License
This guide is released under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png)
