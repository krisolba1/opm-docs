# opm-docs
Documentation on how to set up a local copy of opm web services on a clean Debian 9 virtual machine.

This documentation assumes you have a clean virtual machine with debian 9, and relevant backups of wordpress/wiki databases/files and mailman.

# Installation lemp

## Nginx
Run: 
- *sudo apt-get update* 
- *sudo apt-get install nginx*
    
Open as root /etc/nginx/sites-available/default
- Make sure you have index.php as the first value instead of index.html
- Uncomment the php block: location ~\.php$ { }
- Make root of document /usr/share/nginx/wordpress;
- Uncomment .htaccess block
- Include the following block:

```javascript
    location /package {
		alias /usr/share/nginx/package/;
		autoindex on;
	}
	
	location /plots {
		alias /usr/share/nginx/plots/;
		autoindex on;
	}
	
	location /apidoc {
		alias /var/www/apidocumentation/;
		autoindex on;
		
		location ~ \.php$ {
			include fastcgi_params;
			fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
			fastcgi_split_path_info ^(.+.php)(/.*)$;
			fastcgi_index index.php;
			fastcgi_param SCRIPT_FILENAME $request_filename;
		}
	}

	location /forms/ {
		alias /usr/share/nginx/html/;
		autoindex on;
	}
```
- Run: *sudo nginx -t* to check syntax, restart nginx
    
## php 
- Run: *sudo apt-get install php-fpm php-mysql php7.0 php7.0-fpm php7.0-curl php7.0-gd php7.0-mbstring php-xml* 
- Open as root /etc/php/7.0/fpm/php.ini (f.ex vim) 
  - Search for cgi.fix_pathinfo=0, uncomment this line and change value 0 to 1. 
  - Run: *sudo systemctl restart php7.0-fpm*
- Test configuration php/nginx
- Go to document root folder (/usr/share/nginx  /var/www/html or whatever the root is)
  - *sudo vim info.php*, insert this line: <?php phpino();, save and close file.
-  Go to your.webaddres/info.php (if conf is ok, this should show php info page. Remove info.php if succesful).


## mysql
- Run: *sudo apt-get install mariadb-server mariadb-client php7.0-mysql*
- Run: *sudo systemctl restart php7.0-fpm.service*
- Run: *sudo mysql\
           MariaDB> use mysql;\
           MariaDB> update user set plugin='' where User='root';\
           MariaDB> flush privileges;\
           MariaDB> exit*
- Run: *mysql_secure_installation* -> set root password and answer the questions.

## ssh
Create ssh keys etc
- mkdir .ssh
- ssh-keygen -t rsa
- sudo apt-get install netcat-openbsd
Create config file .ssh/config and insert the following:\
*Host opm-project.org  opm-backup.northeurope.cloudapp.azure.com\
      ProxyCommand=netcat -X connect -x www-proxy.statoil.no:80 %h %p\
      ServerAliveInterval 10*

Copy ssh public key to homedir on opm-project.org  and to backup server (opm-backup)
- ssh-copy-id kristin@opm-project.org
- ssh-copy-id opm-backup@opm-backup.northeurope.cloudapp.azure.com (or cat id_rsa.pub and paste to opm-backups authorized_keys)
## Wordpress 
### Wordpress files
Copy the wordpress files over from backup to local server. There are daily backup tar files located on backup server: opm-backup@opm-backup.northeurope.cloudapp.azure.com:/datadrive/opm-backups/wordpress/wordpress-files, copy backup over to somewhere your user can access it (e.g /home/kristin) 
- *rsync -av opm-backup@opm-backup.northeurope.cloudapp.azure.com:/datadrive/opm-backups/wordpress/wordpress-files/wordpressbackup.tar.gz /home/kristin* 
- *sudo tar xzvf /home/kristin/wordpressbackup.tar.gz --directory /* (This will put the files on local server under /usr/share/nginx/wordpress)

- There is a http version tar file under /datadrive/opm-backups/wordpress/wordpress.http.tar.gz.

Repeat the above for the following folder:
  - /usr/share/nginx/package (location on backup-server /datadrive/opm-backups/package)
  - /usr/share/nginx/plots  (location on backup server /datadrive/opm-backups/plots)
  - /var/www/apidocumentation (location on backup server /datadrive/opm-backups/apidocumentation)
  - /usr/share/nginx/html (location on backup server /datadrive/opm-backups/html)

### Wordpress database
There are db backups on opm-backup:/datadrive/opm-backups/wordpress/wordpress-db. Copy the last one (or another one if desired over to somewhere your user can access it). 
- *rsync -av opm-backup@opm-backup.northeurope.cloudapp.azure.com:/datadrive/opm-backups/wordpress/wordpress-db/wordpress-db-backup.sql /home/kristin*
#### Create database
Log in to mysql
- *mysql -u root -p*
- *mysql> CREATE DATABASE 'dbname';*
- *mysql> USE 'dbname';*
- *mysql> CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';* 
- *mysql> GRANT ALL PRIVILEGES ON * . * TO 'user'@'localhost';*
- *mysql> FLUSH PRIVILEGES;*

Restore the database from backup.sql file (mysql -u "user" -p "databasename"<"backupfile.sql")
- *mysql -u user -p dbname<wordpress-db-backup.sql*

Go to localhost in browser. If you get "error establishing database connection", it's probably an issue with the db backup file, try to restore from another backup file. Since this is a local copy you can test that it is working without internet access as well. 

#### Tips
- Update /etc/hosts -> 127.0.0.1       opm-project.org
- If some files have not been synced, it is probably caused by trouble with permissions, f.ex this could be the case with wp-content/uploads folder. 

## Mediawiki
### Wiki files
Copy the wiki files over from backup to local server. There are daily backup tar files located in /datadrive/opm-backups/mediawiki/wiki-files, copy backup over to somewhere your user can access it (e.g /home/kristin) 
- *rsync -av opm-backup@opm-backup.northeurope.cloudapp.azure.com:/datadrive/opm-backups/mediawiki/wiki-files/wikibackup.tar.gz /home/kristin* 
- *sudo tar xzvf /home/kristin/wikibackup.tar.gz --directory /* (This will put the files on local server under /usr/share/nginx/mediawiki). Make sure the owner is www-data:www-data (not kristin/or username)
- Update nginx config file with new server block:
```javascript
    Server{
            listen 80;
            root /usr/share/nginx/mediawiki;
            index index.php;
            server_name wiki.opm-project.org;
            
            location ~ \.php$ {
                    try_files $uri = 404;
                    fastcgi_pass unix:/run/php/php7.0-fpm.sock;
                    fastcgi_index index.php;
                    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                    include fastcgi_params;
            }
     }
```
Update /etc/hosts  ->  127.0.0.1       wiki.opm-project.org

### Wiki database
There are db backups on opm-backup:/datadrive/opm-backups/mediawiki/mediawiki-db. Copy the last one (or another one if desired over to somewhere your user can access it). 
- *rsync -av opm-backup@opm-backup.northeurope.cloudapp.azure.com:/datadrive/opm-backups/mediawiki/mediawiki-db/wiki-db-backup.sql /home/kristin*
#### Create database
Log in to mysql
- *mysql -u root -p*
- *mysql> CREATE DATABASE dbname;*
- *mysql> USE opmwiki;*
- *mysql> CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';* 
- *mysql> GRANT ALL PRIVILEGES ON * . * TO 'user'@'localhost';*
- *mysql> FLUSH PRIVILEGES;*

Restore the database from backup.sql file (mysql -u "user" -p "databasename"<"backupfile.sql")
- *mysql -u user -p dbname<wiki-db-backup.sql*

## Mailman
Edit nginx config file to include the following
```javascript
    location /cgi-bin/mailman {
		root /usr/lib/;
		fastcgi_split_path_info (^/cgi-bin/mailman/[^/]*)(.*)$;
		include /etc/nginx/fastcgi_params;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		fastcgi_param PATH_INFO $fastcgi_path_info;
		fastcgi_param PATH_TRANSLATED $document_root$fastcgi_path_info;
		fastcgi_intercept_errors on;
		fastcgi_pass unix:/var/run/fcgiwrap.socket;
	}
	location /images/mailman {
		alias /usr/share/images/mailman;
	}
	location /pipermail {
		alias /var/lib/mailman/archives/public/;
		index index.html;
		autoindex on;
	}
```
Folders to keep backup of:
- /usr/lib/mailman (location of backup: /datadrive/opm-backups/mailman/mailman_usr_lib)
- /var/lib/mailman  (location of backup: /datadrive/opm-backups/mailman/mailman_var_lib)
- /etc/mailman  (location of backup: /datadrive/opm-backups/mailman/templates)
- /var/log/mailman (location of backup: /datadrive/opm-backups/mailman/logs)
- /var/lock/mailman (location of backup: /datadrive/opm-backups/mailman/locks)
- /usr/lib/cgi-bin/mailman (location of backup: /datadrive/opm-backups/mailman/cgibin)
- /usr/share/images/mailman (location of backup: /datadrive/opm-backups/mailman/Icons)
There are backups of all these folders on opm-backup.northeurope.cloudapp.azure.com, at /datadrive/opm-backups/mailman

These above folders must be copied from backup to local server.

- For the archives: If some mails are missing from archive, run as root\
    *./bin/arch listname*
- Listinfo: I had no fcgiwrap.socket file, fixed by *sudo apt-get install fcgiwrap*
- Permissions: check permissions *sudo ./bin/check_perms -f*
    

    
