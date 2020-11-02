# cloud-1

### Project Description:
Setup a simple WordPress website on a cloud infrastructure (GCP was used).
### Specifications:
- Website must be running at all times on at least 2 servers.
- Load balancers used to distribute website visitors to different instances.
- Traffic spikes will be handled in the best way (Auto-scaling).
- Logged in users will stay identified for the length of a normal session.
- CDN used to distribute static content.
- Admin privileges should be handled correctly.

## Step-By-Step Process taken for GCP setup:
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
             
2. Create an Image from your WordPress instance:
    1. First stop the current WordPress VM you have been working on.
    2. Go to 'Compute Engine' tab on GCP and select 'Images' from the side panel.
    3. When creating the image set 'Source Disk' as your WordPress VM and 'Location' as regional.
    
3. Create an Instance Template from your Image:
    1. Navigate to 'Instance Templates' tab on GCP.
    2. When creating the template:
        - Change the boot disk to a custom image and select the image you have just created.
        - Check allow HTTP and HTTPS traffic.

4. Create an Instance Group from your Instance Template:
    1. Navigate to 'Instance Groups' tab on GCP:
    2. When creating the group:
        - Set 'Location' as Single Zone.
        - Set instance template as the one you have just created.
        - Set Autoscaling to on.
        - Minimum number of instances to 2.
        - Maximum number of instances to 6.
        - Create a health check.
        
5. Create a firewall rule:
    1. Navigate to VPC Network tab in GCP.
    2. When creating a new firewall rule:
        - Set 'Direction' as Ingress.
        - Set 'Action on Match' as Allow.
        - Set 'Targets' as All Instances.
        - Set 'Source Filter' as IP Range.
        - Set range as `130.211.0.0/22 35.191.0.0/16`.
        - Check TCP on and set it to 80.
        
6. Create the HTTPS Loadbalancer:
    1. Navigate to the 'Network Services' tab in GCP.
    2. When creating a new loadbalancer:
     - Choose HTTP/S loadbalancer.
     - Choose 'From Internet to my VMs'.
     - In backend setup: Set protocol as HTTPS and name it such. Set the instance-group as the group you have
        created previously. Set the port number to 80. Check 'Enable Cloud CDN'. Apply the same health check you
        used for your Instance Group.
     - In frontend setup: Set protocol as HTTPS. Set your IP Address as static by creating a new one and reserving it.
        Port must be 80.
        
7. Create the HTTP Loadbalancer:
    1. Navigate to the 'Network Services' tab in GCP.
    2. When creating a new loadbalancer:
         - Choose HTTP/S loadbalancer.
         - Choose 'From Internet to my VMs'.
         - Do not create a backend.
         - Setup your host and path rules: Check 'Advanced host and path rule (URL redirect, URL rewrite)'.
            Set action as 'Redirect the client...'. Response code should be 300.
            Check 'HTTPS redirect'.
         - In frontend setup: Set protocol as HTTP. Set your IP Address as the static address you are using for the HTTPS Loadbalancer.
            Port must be 80 (same as HTTPS Loadbalancer).