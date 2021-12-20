# How to setup Let's Encrypt for Nginx on Ubuntu 18.04 (including IPv6, HTTP/2 and A+ SLL rating)

-------------------------------------------------------------------------------

## Virtual hosts

Let's say you want to host domains `first.com` and `second.com`.

Create folders for their files:

	mkdir /var/www/first
	mkdir /var/www/second

Create a text file `/etc/nginx/sites-available/first.conf` containing:

	server {
		listen 80 default_server;
		listen [::]:80 default_server ipv6only=on;

		server_name first.com www.first.com;
		root /var/www/first;

		index index.html;
		location / {
			try_files $uri $uri/ =404;
		}
	}

Create a text file `/etc/nginx/sites-available/second.conf` containing:

	server {
		listen 80;
		listen [::]:80;

		server_name second.com www.second.com;
		root /var/www/second;

		index index.html;
		location / {
			try_files $uri $uri/ =404;
		}
	}

Note that **only the first domain** has the keywords `default_server` and `ipv6only=on` in the `listen` lines.

Replace the default virtual host:

	sudo rm /etc/nginx/sites-enabled/default
	sudo ln -s /etc/nginx/sites-available/first.conf /etc/nginx/sites-enabled/first.conf
	sudo ln -s /etc/nginx/sites-available/second.conf /etc/nginx/sites-enabled/second.conf

	sudo nginx -t
	sudo systemctl stop nginx
	sudo systemctl start nginx

Check that Nginx is running:

	sudo systemctl status nginx

Expected results at this stage:
 - `http://first.com` and `http://www.first.com` serve the files from `/var/www/first`
 - `http://second.com` and `http://www.second.com` serve the files from `/var/www/second`
 - `https://www.first.com` and `https://www.second.com` don't work yet


-------------------------------------------------------------------------------

## Certbot

Install Certbot for Nginx:

	sudo apt-get update
	sudo apt-get install software-properties-common
	sudo add-apt-repository universe
	sudo add-apt-repository ppa:certbot/certbot
	sudo apt-get update
	sudo apt-get install certbot python-certbot-nginx

Setup the certificates & convert Virtual Hosts to HTTPS:

	sudo certbot --nginx

It will ask for:
 - an email address
 - agreeing to its Terms of Service
 - which domains to use HTTPS for (it detects the list using `server_name` lines in your Nginx config)
 - whether to redirect HTTP to HTTPS (recommended) or not

**You could stop here if all you want is HTTPS** as this already gives you an `A` rating and maintains itself.

Test your site with SSL Labs using `https://www.ssllabs.com/ssltest/analyze.html?d=www.YOUR-DOMAIN.com`

Expected results at this stage:
 - `http://first.com` redirects to `https://first.com`
 - `http://second.com` redirects to `https://second.com`
 - `http://www.first.com` redirects to `https://www.first.com`
 - `http://www.second.com` redirects to `https://www.second.com`
 - `https://first.com` and `https://www.first.com` serve the files from `/var/www/first`
 - `https://second.com` and `https://www.first.com`serve the files from `/var/www/second`


-------------------------------------------------------------------------------

## Automatic renewal

**There is nothing to do**, Certbot installed a cron task to automatically renew certificates about to expire.

You can [check renewal works](https://certbot.eff.org/docs/using.html#re-creating-and-updating-existing-certificates) using:

	sudo certbot renew --dry-run

You can also [check what certificates exist](https://certbot.eff.org/docs/using.html#managing-certificates) using:

	sudo certbot certificates


-------------------------------------------------------------------------------

## HTTP/2

`first.conf` should now look something like this, now that Certbot edited it:

	server {
		server_name first.com www.first.com;
		root /var/www/first.com;

		index index.html;
		location / {
			try_files $uri $uri/ =404;
		}

		listen [::]:443 ssl ipv6only=on; # managed by Certbot
		listen 443 ssl; # managed by Certbot
		ssl_certificate /etc/letsencrypt/live/first.com/fullchain.pem; # managed by Certbot
		ssl_certificate_key /etc/letsencrypt/live/first.com/privkey.pem; # managed by Certbot
		include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
		ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
	}

	server {
		if ($host = www.first.com) {
			return 301 https://$host$request_uri;
		} # managed by Certbot

		if ($host = first.com) {
			return 301 https://$host$request_uri;
		} # managed by Certbot

		listen 80 default_server;
		listen [::]:80 default_server;

		server_name first.com www.first.com;
		return 404; # managed by Certbot
	}

Certbot didn't add HTTP/2 support when it created the new server blocks, so replace these lines:

	listen [::]:443 ssl ipv6only=on;
	listen 443 ssl;

by this:

	listen [::]:443 ssl http2 ipv6only=on;
	listen 443 ssl http2;
	gzip off;

There is [already an open Github issue](https://github.com/certbot/certbot/issues/3646)
requesting Certbot to add `http2` automatically, so hopefully this step can soon be removed.


-------------------------------------------------------------------------------

## Stronger settings for A+

### Trusted certificate

The HTTPS `server` blocks in `first.conf` and `second.conf` contain these lines, added by Certbot:

	ssl_certificate /etc/letsencrypt/live/YOUR-DOMAIN/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/YOUR-DOMAIN/privkey.pem;

The stronger settings use **OCSP Stapling**, so each virtual host will need a `ssl_trusted_certificate` as well.

**Add this line** (using the folder name that Certbot generated for your domain) after the `ssl_certificate_key` line:

	ssl_trusted_certificate /etc/letsencrypt/live/YOUR-DOMAIN/chain.pem;


---
### SSL

Now let's **edit the shared SSL settings** at `/etc/letsencrypt/options-ssl-nginx.conf`.
It most likely looks like this initially:

	ssl_session_cache shared:le_nginx_SSL:1m;
	ssl_session_timeout 1440m;

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	ssl_prefer_server_ciphers on;
	ssl_ciphers "ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS";

If you tested with SSL Labs, you probably noticed that quite a few ciphers were flagged as "weak".

So **replace the contents of the file** with:

	ssl_session_cache shared:le_nginx_SSL:1m;
	ssl_session_timeout 1d;
	ssl_session_tickets off;

	ssl_protocols TLSv1.2;
	ssl_prefer_server_ciphers on;
	ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
	ssl_ecdh_curve secp384r1;

	ssl_stapling on;
	ssl_stapling_verify on;

	add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload;";
	add_header Content-Security-Policy "default-src 'none'; frame-ancestors 'none'; script-src 'self'; img-src 'self'; style-src 'self'; base-uri 'self'; form-action 'self';";
	add_header Referrer-Policy "no-referrer, strict-origin-when-cross-origin";
	add_header X-Frame-Options SAMEORIGIN;
	add_header X-Content-Type-Options nosniff;
	add_header X-XSS-Protection "1; mode=block";

Now **restart Nginx**, and test the domain again with SSL Labs
using `https://www.ssllabs.com/ssltest/analyze.html?d=www.YOUR-DOMAIN.com&latest`:
it should now be rated `A+`, congratulations! 🙂


-------------------------------------------------------------------------------

## Conclusion

You could further improve using content-specific features like `Content Security Policy` and `Subresource Integrity`, and [Brotli compression](https://caniuse.com/#feat=brotli) to replace *gzip*.


Online testing tools:
 - [Mozilla Observatory](https://observatory.mozilla.org/)
 - [SSL Labs](https://www.ssllabs.com/ssltest/)
 - [Security Headers](https://securityheaders.com)

Useful links:
 - [Subresource Integrity](https://developer.mozilla.org/fr/docs/Web/Security/Subresource_Integrity)
 - [Content Security Policy](https://developer.mozilla.org/fr/Add-ons/WebExtensions/Content_Security_Policy)
 - [Mozilla Security Guidelines](https://wiki.mozilla.org/Security/Guidelines/Web_Security)
 - [Certbot documentation](https://certbot.eff.org/docs/)

If **Let's Encrypt is useful to you**, consider [donating to Let's Encrypt](https://letsencrypt.org/donate/)
or [donating to the EFF](https://supporters.eff.org/donate/).
