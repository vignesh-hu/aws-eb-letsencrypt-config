## [Let's Encrypt](https://letsencrypt.org/) - Enable HTTPS in AWS Elastic Beanstalk website.

### Example usage:

1. Create `.ebextensions` folder in your root of your project.

2. Create `ssl.config` file under `.ebextensions` folder and paste the following code to enable HTTPS to your website. This code has an auto renew enabled so that you don't have to worry about when expiring your SSL Certificate.

```
packages:
  yum:
    mod24_ssl : []

files:
  "/etc/httpd/conf.d/ssl_rewrite.conf":
    mode: "000644"
    owner: root
    group: root
    content: |
       RewriteEngine On
       RewriteCond %{HTTP:X-Forwarded-Proto} !https
       RewriteCond %{HTTP_USER_AGENT} !ELB-HealthChecker
       RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
  /tmp/ssl.conf:
    mode: "000644"
    owner: root
    group: root
    content: |
      LoadModule ssl_module modules/mod_ssl.so
      Listen 443
      <VirtualHost *:443>
        <Proxy *>
          Order deny,allow
          Allow from all
        </Proxy>

        SSLEngine on
        SSLCertificateFile "/etc/letsencrypt/live/ebcert/fullchain.pem"
        SSLCertificateKeyFile "/etc/letsencrypt/live/ebcert/privkey.pem"
        SSLProtocol           All -SSLv2 -SSLv3
        SSLHonorCipherOrder   On
        SSLSessionTickets     Off

        Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains; preload"
        Header always set X-Frame-Options DENY
        Header always set X-Content-Type-Options nosniff

        ProxyPass / http://localhost:80/ retry=0
        ProxyPassReverse / http://localhost:80/
        ProxyPreserveHost on
        RequestHeader set X-Forwarded-Proto "https" early
      </VirtualHost>

  /tmp/certificate_renew:
    mode: "000644"
    owner: root
    group: root
    content: |
      0 0 * * 0 root /opt/certbot/certbot-auto renew --webroot --webroot-path /var/www/html/ --pre-hook "killall httpd" --post-hook "sudo restart supervisord || sudo start supervisord" --force-renew --preferred-challenges http-01 >> /var/log/certificate_renew.log 2>&1
container_commands:
  # creates a swapfile to avoid memory errors in small instances
  00_enable_swap:
    command: "sudo swapoff /tmp/swapfile_certbot ; sudo rm /tmp/swapfile_certbot ; sudo fallocate -l 1G /tmp/swapfile_certbot && sudo chmod 600 /tmp/swapfile_certbot && sudo mkswap /tmp/swapfile_certbot && sudo swapon /tmp/swapfile_certbot"
  # installs certbot
  10_install_certbot:
    command: "mkdir -p /opt/certbot && wget https://dl.eff.org/certbot-auto -O /opt/certbot/certbot-auto && chmod a+x /opt/certbot/certbot-auto"
  # issue the certificate if it does not exist
  20_install_certificate:
    command: "sudo /opt/certbot/certbot-auto certonly --debug --non-interactive --email ${LETSENCRYPT_EMAIL} --agree-tos --webroot --webroot-path /var/www/html/ --domains ${LETSENCRYPT_DOMAIN} --keep-until-expiring --preferred-challenges http-01 --force-renewal"
  # create a link so we can use '/etc/letsencrypt/live/ebcert/fullchain.pem'
  # in the apache config file
  30_link:
    command: "ln -sf /etc/letsencrypt/live/${LETSENCRYPT_DOMAIN} /etc/letsencrypt/live/ebcert"
  # move the apache .conf file to the conf.d directory.
  # Rename the default .conf file so it won't be used by Apache.
  40_config:
    command: "mv /tmp/ssl.conf /etc/httpd/conf.d/ssl.conf && mv /etc/httpd/conf.d/wsgi.conf /etc/httpd/conf.d/wsgi.conf.bkp || true"
  # clear the swap files created in 00_enable_swap
  50_cleanup_swap:
    command: "sudo swapoff /tmp/swapfile_certbot && sudo rm /tmp/swapfile_certbot"
  # kill all httpd processes so Apache will use the new configuration
  60_killhttpd:
    command: "killall httpd ; sleep 3"
  # Add renew cron job to renew the certificate
  70_cronjob_certificate_renew:
    command: "mv /tmp/certificate_renew /etc/cron.d/certificate_renew"
```

3. After that you need to setup two environment variables (eg: LETSENCRYPT_EMAIL, LETSENCRYPT_DOMAIN) in your instance because this is required in the code above.

> `LETSENCRYPT_DOMAIN` - the domain that you want to enable the SSL (eg: mydomain.example.com)

> `LETSENCRYPT_EMAIL` - the email that will be used to notify you about your SSL Certificate (example: letsencrypt will notify you when your SSL will expire or etc.)

That's it once you deploy this, HTTPS is already enabled in your website. Enjoy!
