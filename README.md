# Nginx FastCGI Cache Purge & Preload for Wordpress
![cache_preload](https://user-images.githubusercontent.com/25556606/202007501-8d9e5ab6-3330-452f-b967-6615e703a486.png)<br/>
------
![ıuoyuı](https://user-images.githubusercontent.com/25556606/202256497-15f46225-b06b-4e37-a3b6-1b2c1ff0259b.png)<br/>
![oıu](https://user-images.githubusercontent.com/25556606/202257768-e36986ff-6bfa-4646-befe-60ed3518835a.png)<br/>
![ouoıuy](https://user-images.githubusercontent.com/25556606/202265347-cf901dd7-65d2-4e23-b1d3-ba46ae1ddbcb.png)
-------

Pluginless Nginx cache management solution for wordpress. If you have ngx_cache_purge or nginx_cache_purge modules then some wordpress plugins are available. Check Nginx Helper or Cache Sniper for Nginx. On my side none of them worked as expected so I decided to make my own solution.

## Integration is pretty simple if you are native linux user and managing your own server. Here is the short explanation of proper php-fpm nginx setup --<br/>

#### PHP-FPM-USER (as known as the website user)
The PHP-FPM user should be a special user that you create for running your website, whether it is Magento, WordPress, or anything.

#### WEBSERVER-USER (as known as the webserver user)
NGINX must run with it own unprivileged user, which is **nginx** (RHEL-based systems) or **www-data** (Debian-based systems).

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

```
chmod -R u=rwX,g=rX,o= /path/to/website/files
```
This translates to the following:

- PHP-FPM-USER can read, write all files, and read all directories
- PHP-FPM-GROUP (WEBSERVER-USER) can read all files and traverse all directories, but not write
- All other users cannot read or write anything

This is proper php-fpm nginx setup example. With this short explanation, I think you understand better which user will be owner of the **fastcgi_ops.sh** --> PHP-FPM user (website user) -- NOT nginx or www-data !

### Let's Integrate
Before starting make sure the ACL enabled on your environment. Check **/etc/fstab** and make sure acl is exist. If you don't see **acl** flag ask to google how to enable ACL on linux.

```
/dev/sda3 / ext4 noatime,errors=remount-ro,acl 0 1
```

1) put **fastcgi_ops.sh** to website user's home. e.g. **/home/php-fpm-user/scripts** (avoid to web root directory)<br/>
2) change ownership of the script to the **website user** via **chown php-fpm-user:php-fpm-group fastcgi_ops.sh**<br/>
3) make script executable via **chmod +x fastcgi_ops.sh**<br/>
4) set your fastcgi cache path, preload domain, website user and mail options on **fastcgi_ops.sh**<br/>
5) open systemd service file (**wp-fcgi-notify.service**) and set execstart & stop script path e.g. **/home/php-fpm-user/scripts/fastcgi_ops.sh** (keep the script arguments **--wp-inotify-start** and **--wp-inotify-stop**)<br/>
6) open **functions.php** and set script path e.g. **/home/php-fpm-user/scripts/fastcgi_ops.sh**<br/>
7) get functions.php codes and add to your **child theme's functions.php**<br/>
8) move **wp-fcgi-notify.service** to **/etc/systemd/system/** and start service **under root**. Check service is started without any error.
```
cp wp-fcgi-notify.service /etc/systemd/system/
systemctl daemon-reload
systemctl start wp-fcgi-notify.service
systemctl enable wp-fcgi-notify.service
```
> ##### Why we need systemd service under root?
> Things get a little messy here. We have two user, WEBSERVER-USER and PHP-FPM-USER and cache purge operations will be done by the PHP-FPM-USER but nginx always creates the cache folder&files with the WEBSERVER-USER (with very strict permissions).
Because of strict permissions adding PHP-FPM-USER to WEBSERVER-GROUP not solve the permission problems. The thing is even you set recursive default setfacl for cache folder, Nginx always override it, surprisingly Nginx cache isn't following ACLs also. Executing scripts with sudo privilege is not possible because PHP-FPM-USER is not sudoer. (it shouldn't be). So how we will purge nginx cache with PHP-FPM-USER? Combining **inotifywait** with **setfacl** under root is only solution that I found. **wp-fcgi-notify.service** listens fastcgi cache folder for **create** events with help of **fastcgi_ops.sh** and give write permission recursively to PHP-FPM-USER for further purge operations.
