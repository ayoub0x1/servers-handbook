# My Servers Handbook

                                
**My notes on servers administration, basics, security, tips & tricks. This was supposed to be personal but I'm making it public in case anyone needs something from it or want contribute to it.**


# Mongodb

**Find the proccesses running on same mongod port in case of conflict issue:**

`sudo lsof -iTCP -sTCP:LISTEN -n -P | grep mongod`

**Secure the database on production:**

Mongodb comes with a lot of security options turned off by default, so here are a few things that you must ensure to do if you've a new installation and you're pushing to production.

`$ mongo`

```
> use admin
> db.createUser(
>  {
>    user: "Kei",
>    pwd: "Kei'sSecurePassword",
>    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
>  }
> )
```


**Login after securing:**

`mongo -u Kei -p --authenticationDatabase admin`


**Changing the default settings to more secure ones:**

You will find the configuration file at `/etc/mongodb.conf`

To change the default port mongodb is using, uncomment `#port = 27017` and change it to the one you like.

Uncomment `#auth = true` to enable authentication.


# NodeJS and Nginx

**Set Up Nginx as a Reverse Proxy Server**

Add this to your server block configuration.

```
    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
You can add additional location blocks to the same server block to provide access to other applications on the same server. 
For example, if you were also running another Node.js application on port `8081`, you could add this location block to allow access to it via `http://example.com/app2`

```
 location /app2 {
        proxy_pass http://localhost:8081;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
```

**PS:** The default configuration comes with a 404 setup on the location block, if you keep it there it won't serve all assets. So if your application runs on http://example.com/app, anything after that will return `Not Found`.
You can remove it and set it somewhere else to serve everything.

# Securing Nginx

**Hide Nginx version number:**

`server_tokens off;`

**Hide Nginx server signature:**

`more_set_headers "Server: Unknown";`

**Hide upstream proxy headers:**

```
proxy_hide_header X-Powered-By;
proxy_hide_header X-AspNetMvc-Version;
proxy_hide_header X-AspNet-Version;
proxy_hide_header X-Drupal-Cache;
```

**Force all connections over TLS:**

TLS provides two main services. For one, it validates the identity of the server that the user is connecting to for the user. It also protects the transmission of sensitive information from the user to the server.

This can be done easily by using Letsencrypt, if you wish for it, it will handle redirecting all the traffic to HTTPS.

**HTTP Strict Transport Security:**

You must ensure your website is indeed all HTTPS before you use this header. 
Generally HSTS is a way for websites to tell browsers that the connection should only ever be encrypted. This prevents MITM attacks, downgrade attacks, sending plain text cookies and session ids.
The header indicates for how long a browser should unconditionally refuse to take part in unsecured HTTP connection for a specific domain.

`add_header Strict-Transport-Security "max-age=63072000; includeSubdomains" always;`

**CSP (Content Security Policy):**

CSP reduce the risk and impact of XSS attacks in modern browsers. The key word here is reduce, by no means it prevents XSS from happening. CSP is a good defence-in-depth measure to make exploitation of an accidental lapse in that less likely.

```
# This policy allows images, scripts, AJAX, and CSS from the same origin, and does not allow any other resources to load.
add_header Content-Security-Policy "default-src 'none'; script-src 'self'; connect-src 'self'; img-src 'self'; style-src 'self';" always;
```

**Clickjacking protection (X-Frame-Options):**

`add_header X-Frame-Options "SAMEORIGIN" always;`

**Prevent some categories of XSS (X-XSS-Protection):**

This enables the cross-site scripting filter built into modern web browsers.

`add_header X-XSS-Protection "1; mode=block" always;`

**Preventing mime based attacks:**

`add_header X-Content-Type-Options "nosniff" always;`

**Mitigating Slow HTTP DoS attacks:**

You can close connections that are writing data too infrequently, which can represent an attempt to keep connections open as long as possible (thus reducing the server’s ability to accept new connections).

```
client_body_timeout 10s;
client_header_timeout 10s;
keepalive_timeout 5s 5s;
send_timeout 10s;
```

**Fail2ban Blacklist for offenders:**

`cd /etc/fail2ban/filter.d && sudo wget https://raw.githubusercontent.com/mitchellkrogza/Fail2Ban-Blacklist-JAIL-for-Repeat-Offenders-with-Perma-Extended-Banning/master/filter.d/blacklist.conf -O blacklist.conf`

`cd /etc/fail2ban/action.d && sudo wget https://raw.githubusercontent.com/mitchellkrogza/Fail2Ban-Blacklist-JAIL-for-Repeat-Offenders-with-Perma-Extended-Banning/master/action.d/blacklist.conf -O blacklist.conf`

Open `/etc/fail2ban/jail.local` (Assuming you've already made a copy from the original Fail2ban jail file)

Add this to the bottom of the file:

```
[blacklist]
enabled = true
logpath  = /var/log/fail2ban.*
filter = blacklist
banaction = blacklist
bantime  = 31536000; 1 year
findtime = 31536000; 1 year
maxretry = 10
```

Now make the blacklist file in fil2ban directory and make it writable. 

`sudo touch /etc/fail2ban/ip.blacklist && sudo chmod 755 /etc/fail2ban/ip.blacklist`


# Nginx Tricks

**Making one error page for everything.**

First of all you need to create an error_page in your http, server, or location directive, the location block should be part of the server directive:

```
error_page 400 401 402 403 404 405 406 407 408 409 410 411 412 413 414 415 416 417 418 421 422 423 424 426 428 429 431 451 500 501 502 503 504 505 506 507 508 510 511 /error.html;

location = /error.html {
  ssi on;
  internal;
  root /var/www/html;
}
```



After that you create a error.html file in `/var/www/html/`. Next, you map the error codes.

To get the variable status_text to work you will need this map in your the http directive [(docs):](https://nginx.org/en/docs/http/ngx_http_map_module.html)

```
map $status $status_text {
  400 'Bad Request';
  401 'Unauthorized';
  402 'Payment Required';
  403 'Forbidden';
  404 'Not Found';
  405 'Method Not Allowed';
  406 'Not Acceptable';
  407 'Proxy Authentication Required';
  408 'Request Timeout';
  409 'Conflict';
  410 'Gone';
  411 'Length Required';
  412 'Precondition Failed';
  413 'Payload Too Large';
  414 'URI Too Long';
  415 'Unsupported Media Type';
  416 'Range Not Satisfiable';
  417 'Expectation Failed';
  418 'I\'m a teapot';
  421 'Misdirected Request';
  422 'Unprocessable Entity';
  423 'Locked';
  424 'Failed Dependency';
  426 'Upgrade Required';
  428 'Precondition Required';
  429 'Too Many Requests';
  431 'Request Header Fields Too Large';
  451 'Unavailable For Legal Reasons';
  500 'Internal Server Error';
  501 'Not Implemented';
  502 'Bad Gateway';
  503 'Service Unavailable';
  504 'Gateway Timeout';
  505 'HTTP Version Not Supported';
  506 'Variant Also Negotiates';
  507 'Insufficient Storage';
  508 'Loop Detected';
  510 'Not Extended';
  511 'Network Authentication Required';
  default 'Something is wrong';
}
```

Test if your config is correct with `sudo nginx -t` then reload it using `sudo nginx -s reload`.


# Hardening Server Tips

**Hiding Linux version from SSH service:**

Add the following line to `/etc/ssh/sshd_config`

`DebianBanner no`

Then restart with `sudo systemctl restart ssh` or `sudo service ssh restart`

**Deactivating unused functions in SSH:**

To prevent unused functions from being exploited, they should be switched off. To apply the setting, the following changes in the SSH configuration file are necessary:

```
AllowTcpForwarding no                   # Disables port forwarding.
X11Forwarding no                        # Disables remote GUI view.
AllowAgentForwarding no                 # Disables the forwarding of the SSH login.
AuthorizedKeysFile .ssh/authorized_keys # The ".ssh/authorized_keys2" file should be removed.
```

**Automatic session timeout:**

With this setting, a forced disconnection of the SSH connection is performed after a certain inactivity. Make the following changes to ssh config:

```
ClientAliveInterval 300
ClientAliveCountMax 1
```



# Tricks & General notes

One of the first things I do when I get a new server is install ohmyzsh, but whenever I have multiple servers running in same time, it gets confusing to tell which server you are on - since zsh only shows a `~` mark instead of the user@host.
If you want to still have that show on ohmyzsh, add this to your `.zshrc` file.

`PROMPT="%{$fg[white]%}%n@%{$fg[green]%}%m%{$reset_color%} ${PROMPT}"`