# cloud-1

##### Project Description:
Setup a simple WordPress website on a cloud infrastructure (GCP was used).
##### Specifications:
- Website must be running at all times on at least 2 servers.
- Load balancers used to distribute website visitors to different instances.
- Traffic spikes will be handled in the best way (Auto-scaling).
- Logged in users will stay identified for the length of a normal session.
- CDN used to distribute static content.
- Admin privileges should be handled correctly.

#### Step-By-Step Process taken for GCP setup:
1. Set-up WordPress instance:
    1. Create WordPress VM instance from Marketplace.
    2. Edit the footer of your WordPress frontpage (2017 theme):
        - SSH into WordPress instance. `sudo su` for super user access.
        - `vim /var/www/html/wp-content/themes/twentyseventeen/template-parts/footer/site-info.php` to edit 'site-info.ph' file.
        - Add to line 22: `printf( __( 'Proudly powered by %s<br>%s', 'twentyseventeen' ), 'WordPress', $_SERVER['SERVER_ADDR']);`
    3. Connect a bucket:
        - Access the url `external_ip_of_wordpress_instance/wp-admin`
        - Access plugins tab and install 'WP Offload Media Lite'.
        - Create a new bucket on GCP under the tab 'Storage'.
        - Creat a new service account under GCP tab 'IAM & Admin'. Download keyfile related to account.
        - SSH into WordPress instance.
        - `sudo su` to have super user access.
        - `cd /var/www/html` to navigate to the directory.
        - Add the following to the 'wp-config.php' file to add the keyfile path:
            `define( 'AS3CF_SETTINGS', serialize( array(
                         'provider' => 'gcp',
                             'key-file-path' => '/etc/file.json',
                     ) ) );`
        - Now navigate to `/etc/` from root.
        - Create the file 'file.json' with the downloaded keyfile text copied into it.
        - In 'WP Offload Media Lite' settings select the bucket you have just created as the option.
    4. Connect an SQL database:
        - Create a new SQL instance on GCP and enable private IP for it.
        - Create a new SQl database in the new instance with charset as 'utf8mb4' and collate as 'utf8mb4_general_ci'.
        - Create a new SQL database user in the new instance and check 'Allow any host' (note your users name and password).
        - SSH into your VM instance and edit the 'wp-config.php' file according to the following:
            `define( 'DB_NAME', 'new sql database name' );
             define( 'DB_USER', 'new sql database user name' );
             define( 'DB_PASSWORD', 'new sql database user password' );
             define( 'DB_HOST', 'private ip address of new sql instance' );
             define( 'DB_CHARSET', 'utf8mb4' );
             define( 'DB_COLLATE', 'utf8mb4_general_ci' );`
    5. Add https rules:
        - Add the following definition to 'wp-config.php' file:
            `define('FORCE_SSL_ADMIN', true);
             if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false)
                     $_SERVER['HTTPS']='on';`
        - Create a file called '.htaccess' at path '/var/www/html'.
        - Add the following rule to the '.htaccess' file:
            `RewriteEngine On
             RewriteCond %{HTTP:X-Forwarded-Proto} =http
             RewriteRule .* https://%{HTTP:Host}%{REQUEST_URI} [L,R=permanent] `
             
2. Create an image from WordPress instance: