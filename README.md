# imap-postgresql-ubuntu2204
## Install an IMAP email server on Ubuntu 22.04 using Postfix, Dovecot, and PostgreSQL using Certbot and fail2ban

These instructions were adopted from [Configure an Email Server with Postfix, Dovecot, and MySQL on Debian and Ubuntu](https://www.linode.com/docs/guides/email-with-postfix-dovecot-and-mysql/)

### Step 1: Install PostgreSQL, Postfix with PGSQL support, and Dovecot w/IMAP and LMTPD support

`sudo apt-get install postgresql postgresql-contrib postfix postfix-pgsql dovecot-core dovecot-imapd dovecot-lmtpd dovecot-pgsql fail2ban`

`sudo snap install --classic certbot`

### Step 1b: Follow website for instructions to install SSL certificate

[Certbot](https://certbot.eff.org/)

### Step 2: Create PostgreSQL database, mailuser, and tables

`sudo -u postgres psql`

`CREATE DATABASE mailserver;`

`CREATE ROLE mailuser WITH PASSWORD 'secure password';`

`ALTER USER mailuser LOGIN;`

`\c mailserver;`

`CREATE TABLE virtual_domains ( id SERIAL PRIMARY KEY, name VARCHAR ( 50 ) NOT NULL );`

`CREATE TABLE virtual_users ( id SERIAL PRIMARY KEY, domain_id integer NOT NULL, password VARCHAR (106) NOT NULL, email VARCHAR (100) UNIQUE NOT NULL, CONSTRAINT fk_domain_id FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE );`

`CREATE TABLE virtual_aliases ( id SERIAL PRIMARY KEY, domain_id integer NOT NULL, source VARCHAR (100) NOT NULL, destination VARCHAR (100) NOT NULL, FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE);`

`GRANT SELECT ON TABLE virtual_users TO mailuser;`

`GRANT SELECT ON TABLE virtual_domains TO mailuser;`

`GRANT SELECT ON TABLE virtual_aliases TO mailuser;`

### Step 3: Insert your domain(s) into virtual_domains table

`INSERT INTO virtual_domains (name) VALUES ('example.com');`

repeat if necessary
verify

`SELECT * FROM virtual_domains;`

### Step 4: Add email addresses and passwords

`python -c 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))'` or `openssl passwd -6`

`INSERT INTO virtual_users (domain_id, password , email) VALUES ('1', 'hash', 'user@example.com');`

`SELECT * FROM mailserver.virtual_users;`

### Step 5: Add any email aliases

`INSERT INTO virtual_aliases (domain_id, source, destination) VALUES ('1', 'alias@example.com', 'user@example.com');`

`SELECT * FROM mailserver.virtual_aliases;`

### Step 6: Configure Postfix MTA email server

https://www.linode.com/docs/guides/email-with-postfix-dovecot-and-mysql/#postfix

```
# Virtual domains, users, aliases, mailbox size limit, and email message size limit in /etc/postfix/main.cf file

# Set Virtual Mailbox limit of 10GB for example
virtual_mailbox_limit = 10000000000
mailbox_size_limit = 10000000000

virtual_mailbox_domains = pgsql:/etc/postfix/pgsql-virtual-mailbox-domains.cf
virtual_mailbox_maps = pgsql:/etc/postfix/pgsql-virtual-mailbox-maps.cf
virtual_alias_maps = pgsql:/etc/postfix/pgsql-virtual-alias-maps.cf, pgsql:/etc/postfix/pgsql-virtual-email2email.cf

# Set email message size limit of 25MB for example
message_size_limit = 25000000
```

### Step 7: Test Postfix

`sudo postmap -v -q example.com pgsql:/etc/postfix/pgsql-virtual-mailbox-domains.cf`
should return 1

`sudo postmap -v -q user@example.com pgsql:/etc/postfix/pgsql-virtual-mailbox-maps.cf`
should retrieve row

`sudo postmap -v -q alias@example.com pgsql:/etc/postfix/pgsql-virtual-alias-maps.cf`
should retrieve row

### Step 8: Setup Postfix Master Settings

[Master Program Settings](https://www.linode.com/docs/guides/email-with-postfix-dovecot-and-mysql/#master-program-settings)

### Step 9: Configure Dovecot

**Note:** In Step 9 on website, change driver to PostgreSQL below.
```
File: /etc/dovecot/dovecot-sql.conf.ext
driver = pgsql
```
[Dovecot](https://www.linode.com/docs/guides/email-with-postfix-dovecot-and-mysql/#dovecot)

```
File: /etc/dovecot/conf.d/10-logging.conf
log_path = /var/log/dovecot.log
info_log_path = /var/log/dovecot-info.log
debug_log_path = /var/log/dovecot-debug.log
auth_verbose = yes
auth_verbose_passwords = sha1
```

### Step 10: Configure fail2ban
```
File: /etc/fail2ban/jail.local
[dovecot]
enabled = true
port    = imap,imaps,submission,465,sieve
logpath = /var/log/dovecot-info.log
backend = %(dovecot_backend)s
[DEFAULT]
bantime = 600
bantime.increment = true
bantime.factor = 1
maxretry = 3
```

```
File: /etc/fail2ban/filter.d/dovecot.conf
prefregex = ^%(__prefix_line)s(?:%(_auth_worker)s(?:\([^\)]+\))?: )?(?:%(__pam_auth)s(?:\(dovecot:auth\))?: |(?:pop3|imap)-login: )?(?:Info: )?<F-CONTENT>.+</F-CONTENT>$

failregex = ^authentication failure; logname=<F-ALT_USER1>\S*</F-ALT_USER1> uid=\S* euid=\S* tty=dovecot ruser=<F-USER>\S*</F-USER> rhost=<HOST>(?:\s+user=<F-ALT_USER>\S*</F-ALT_USER>)?\s*$
            ^(?:Aborted login|Disconnected)(?::(?: [^ \(]+)+)? \((?:auth failed, \d+ attempts(?: in \d+ secs)?|tried to use (?:disabled|disallowed) \S+ auth|proxy dest auth failed)\):(?: user=<<F-USER>[^>]*</F-USER>>,)?(?: method=\S+,)? rip=<HOST>(?:[^>]*(?:, session=<\S+>)?)\s*$
            ^pam\(\S+,<HOST>(?:,\S*)?\): pam_authenticate\(\) failed: (?:User not known to the underlying authentication module: \d+ Time\(s\)|Authentication failure \(password mismatch\?\)|Permission denied)\s*$
            ^[a-z\-]{3,15}\(\S*,<HOST>(?:,\S*)?\): (?:unknown user|invalid credentials|Password mismatch)\s*\(SHA1 of given password:\s*\w*\)$
            <mdre-<mode>>

mdre-aggressive = ^(?:Aborted login|Disconnected)(?::(?: [^ \(]+)+)? \((?:no auth attempts|disconnected before auth was ready,|client didn't finish \S+ auth,)(?: (?:in|waited) \d+ secs)?\):(?: user=<[^>]*>,)?(?: method=\S+,)? rip=<HOST>(?:[^>]*(?:, session=<\S+>)?)\s*$
mode = normal
```
