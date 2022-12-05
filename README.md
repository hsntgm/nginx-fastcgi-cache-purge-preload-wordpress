# Nginx FastCGI Cache Purge & Preload for Wordpress
![cache_preload](https://user-images.githubusercontent.com/25556606/202007501-8d9e5ab6-3330-452f-b967-6615e703a486.png)<br/>
------
![ıuoyuı](https://user-images.githubusercontent.com/25556606/202256497-15f46225-b06b-4e37-a3b6-1b2c1ff0259b.png)<br/>
![oıu](https://user-images.githubusercontent.com/25556606/202257768-e36986ff-6bfa-4646-befe-60ed3518835a.png)<br/>
![ouoıuy](https://user-images.githubusercontent.com/25556606/202265347-cf901dd7-65d2-4e23-b1d3-ba46ae1ddbcb.png)
-------

Pluginless Nginx cache management solution for wordpress. If you have ngx_cache_purge or nginx_cache_purge modules then some wordpress plugins are available. Check Nginx Helper or Cache Sniper for Nginx. On my side none of them worked as expected so I decided to make my own solution.

## Integration is pretty simple if you are native linux user and manage your own server --<br/>

#### PHP-FPM-USER (as known as the website user)
The PHP-FPM user should be a special user that you create for running your website, whether it is Magento, WordPress, or anything.
Its characteristics are the following:

- This is the user that PHP-FPM will execute scripts with
- This user must be unique.
- Do not reuse an existing user account.
- Create a separate user for each website!
- Do not reuse any sudo capable users.
- If your website user is ubuntu or centos, or, root – you’re asking for much trouble.
- Do not use www-data or nginx as website user. This is wrong and will lead to more trouble!
- The username should reflect either the domain name of the website that it “runs”, or the type of corresponding CMS, e.g. magento for a Magento website; or example for example.com website.

#### The WEBSERVER-USER
NGINX must run with it own unprivileged user, which is **nginx** (RHEL-based systems) or **www-data** (Debian-based systems).

This is the “global” webserver user that is used for all websites. So the configuration is straightforward and translates to the following directives in /etc/nginx/nginx.conf:
```
user nginx;
```

#### Connecting PHP-FPM-USER and WEBSERVER-USER
We must connect things up so that WEBSERVER-USER can read files that belong to the PHP-FPM-GROUP<br/>
This will allow us to control what WEBSERVER-USER can read or not, via group chmod permission bit.
```
usermod -a -G PHP-FPM-GROUP WEBSERVER-USER
```
This reads as: add WEBSERVER-USER to PHP-FPM-GROUP.<br/>

```
chown -R PHP-FPM-USER:PHP-FPM-GROUP /path/to/website/files
```
Here is a simple rule: all the files should be owned by the PHP-FPM-USER and the PHP-FPM-USER’s group:

**Incorrect chown examples:**

- www-data:www-data
- nginx:nginx

**Correct chown example:**

- PHP-FPM-USER:PHP-FPM-GROUP

```
chmod -R u=rwX,g=rX,o= /path/to/website/files
```
This translates to the following:

- PHP-FPM-USER can read, write all files, and read all directories
- PHP-FPM-GROUP (WEBSERVER-USER) can read all files and traverse all directories, but not write
- All other users cannot read or write anything

This is proper fpm nginx setup example. With this short explanation, I think you understand better which user will be owner of the bash scripts --> PHP-FPM user (website user) -- NOT nginx or www-data !

### Let's Integrate
1) put all scripts to **same folder** (home folder of the PHP-FPM-USER for example under **/home/PHP-FPM-USER/scripts** (except web root directory), change ownership of scripts to the **PHP-FPM-USER** and make all scripts executable via **chmod +x**<br/>
2) set your fastcgi cache path, preload domain, web site user and mail options on each script<br/>
3) run **fastcgi_inotify.sh** under **root** once and check any error occurs, if everything is ok put fastcgi_inotify.sh to **root crontab**
```
@reboot /home/PHP-FPM-USER/scripts/fastcgi_inotify.sh
```
> ##### Why we need to run fastcgi_inotify.sh under root?
> Things get a little messy here. We have two user, WEBSERVER-USER and PHP-FPM-USER and cache purge operations will be done by the PHP-FPM-USER but nginx always creates the cache folder&files with the WEBSERVER-USER (with very strict permissions).
Because of strict permissions adding PHP-FPM-USER to WEBSERVER-GROUP not solve the permission problems. The thing is even you set recursive default setfacl for cache folder, Nginx always override it, surprisingly Nginx cache isn't following ACLs also. Executing scripts with sudo privilege is not possible because PHP-FPM-USER is not sudoer. (it shouldn't be). So how we will purge nginx cache with PHP-FPM-USER? Combining inotifywait with setfacl under root is only solution that I found. **fastcgi_inotify.sh** listens fastcgi cache folder for **create** events and give write permission to PHP-FPM-USER for further purge operations.
4) get functions.php codes, set scripts paths and add to your child theme's functions.php<br/>
