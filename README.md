# Non-public facing Let's Encrypt Certification
This guide details how to setup a Let's Encrypt SSL Certification on a server which isn't accessible for the internet (Non-public facing server or Intranet Server).

## Prerequisites
- Having a public domain name (from now own we will consider the name example.com)
- Choosing a name for the subdomain of that intranet server (we will choose local.example.com)
- Certbot installed on the intranet server
- The intranet server must have a static local ip address (IPv4 or IPv6) or a defined name on the local network DNS server (Hostname)
- The intranet server doesn't have to be a web server, it can be a SFTP server, a MySQL database, any technology that uses SSL/TLS certificates. In this example, we will assume that the server is a web server running on Apache.

## Create the subdomain
On whatever domain registrar you're using, create a new DNS record.
The type must be:
- **A**: if you want to use the local IPv4 server's address
- **AAAA**: for the IPv6 address
- **CNAME**: when using the Hostname

In this example we'll be using a type A record which looks like this:
```
local 1800 IN A 192.168.1.200
```
Of course, you usually don't have to write the line directly and you can use whatever form the registrar provides you.

## Setting the server
On the server (in this example, it's running on Debian 10) add this script to `/etc/apache2/sites-available/default-le-ssl.conf` (or whatever file you're using for the VirtualHost):
```
<VirtualHost *:443>
        ServerName local.example.com
        ...
        #SSLCertificateFile /etc/letsencrypt/live/local.example.com/fullchain.pem
        #SSLCertificateKeyFile /etc/letsencrypt/live/local.example.com/privkey.pem
</VirtualHost>
```
It is important to keep those SSLCertificate lines commented for now as the files have not been generated yet.
You can also add this to automatically redirect HTTP requests:
```
<VirtualHost *:80>
        ServerName local.example.com
        RewriteEngine on
        RewriteCond %{SERVER_NAME} =local.example.com
        RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```
After saving and using the command `sudo apachectl restart`, you can try accessing local.example.com from a web browser. It should tell you that there isn't a valid certificate.

## Using Certbot
On the intranet server use this command `certbot --manual --preferred-challenges dns certonly`.
It should ask you for the domain name(s). Enter: `local.example.com`
It should then generate a challenge key similar to this one: `baDeeI2lEC9vVeUl__zj23sET5x5UN_4h08--9u-98M`
Go to your registrar’s website once more and create a DNS record of type TXT. The name of the record must be `_acme-challenge.local` (local is the subdomain used in this example), and copy-paste the challenge key as its value. The line should look something like this:
```
_acme-challenge.local 1800 IN TXT "baDeeI2lEC9vVeUl__zj23sET5x5UN_4h08--9u-98M"
```
Once this record is active (it can take a few minutes), go back to Certbot and press `Enter`.
The challenge should be successfully verified, and the certificate created. On Debian, Certbot should also automatically schedule the renew processes.
Now edit `/etc/apache2/sites-available/default-le-ssl.conf` and uncomment the SSLCertificate lines. Use the command `sudo apachectl restart` again and that's it.
