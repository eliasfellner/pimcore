# Apache Configuration 

## .htaccess

All the necessary rewrite rules, which are needed for Pimcore to work, must be defined in the `.htaccess` file on path `public/.htaccess`,
as below:

```apacheconf
# Use the front controller as index file. It serves as a fallback solution when
# every other rewrite/redirect fails (e.g. in an aliased environment without
# mod_rewrite). Additionally, this reduces the matching process for the
# start page (path "/") because otherwise Apache will apply the rewriting rules
# to each configured DirectoryIndex file (e.g. index.php, index.html, index.pl).
DirectoryIndex index.php

# By default, Apache does not evaluate symbolic links if you did not enable this
# feature in your server configuration. Uncomment the following line if you
# install assets as symlinks or if you experience problems related to symlinks
# when compiling LESS/Sass/CoffeScript assets.
# Options FollowSymlinks

# Disabling MultiViews prevents unwanted negotiation, e.g. "/index" should not resolve
# to the front controller "/index.php" but be rewritten to "/index.php/index".
<IfModule mod_negotiation.c>
    Options -MultiViews
</IfModule>

# mime types
AddType video/mp4 .mp4
AddType video/webm .webm
AddType image/webp .webp
AddType image/jpeg .pjpeg

Options +SymLinksIfOwnerMatch

# Use UTF-8 encoding for anything served text/plain or text/html
AddDefaultCharset utf-8

RewriteEngine On

<IfModule mod_headers.c>
    <FilesMatch "\.(jpe?g|png)$">
        Header always unset X-Content-Type-Options
    </FilesMatch>
</IfModule>

# Determine the RewriteBase automatically and set it as environment variable.
# If you are using Apache aliases to do mass virtual hosting or installed the
# project in a subdirectory, the base path will be prepended to allow proper
# resolution of the index.php file and to redirect to the correct URI. It will
# work in environments without path prefix as well, providing a safe, one-size
# fits all solution. But as you do not need it in this case, you can comment
# the following 2 lines to eliminate the overhead.
RewriteCond %{REQUEST_URI}::$1 ^(/.+)/(.*)::\2$
RewriteRule ^(.*) - [E=BASE:%1]

# Sets the HTTP_AUTHORIZATION header removed by Apache
RewriteCond %{HTTP:Authorization} .
RewriteRule ^ - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

# Redirect to URI without front controller to prevent duplicate content
# (with and without `/index.php`). Only do this redirect on the initial
# rewrite by Apache and not on subsequent cycles. Otherwise we would get an
# endless redirect loop (request -> rewrite to front controller ->
# redirect -> request -> ...).
# So in case you get a "too many redirects" error or you always get redirected
# to the start page because your Apache does not expose the REDIRECT_STATUS
# environment variable, you have 2 choices:
# - disable this feature by commenting the following 2 lines or
# - use Apache >= 2.3.9 and replace all L flags by END flags and remove the
#   following RewriteCond (best solution)
RewriteCond %{ENV:REDIRECT_STATUS} ^$
RewriteRule ^index\.php(?:/(.*)|$) %{ENV:BASE}/$1 [R=301,L]

<IfModule mod_status.c>
    RewriteCond %{REQUEST_URI} ^/(fpm|server)-(info|status|ping)
    RewriteRule . - [L]
</IfModule>

# restrict access to dotfiles
RewriteCond %{REQUEST_FILENAME} -d [OR]
RewriteCond %{REQUEST_FILENAME} -l [OR]
RewriteCond %{REQUEST_FILENAME} -f
RewriteRule /\.|^\.(?!well-known/) - [F,L]

# ASSETS: check if request method is GET (because of WebDAV) and if the requested file (asset) exists on the filesystem, if both match, deliver the asset directly
RewriteCond %{REQUEST_METHOD} ^(GET|HEAD)
RewriteCond %{DOCUMENT_ROOT}/var/assets%{REQUEST_URI} -f
RewriteRule ^(.*)$ /var/assets%{REQUEST_URI} [PT,L]

# Thumbnails
RewriteCond %{REQUEST_URI} .*/(image|video)-thumb__[\d]+__.*
RewriteCond %{DOCUMENT_ROOT}/var/tmp/thumbnails%{REQUEST_URI} -f
RewriteRule ^(.*)$ /var/tmp/thumbnails%{REQUEST_URI} [PT,L]

# static pages
RewriteCond %{REQUEST_METHOD} ^(GET|HEAD)
RewriteCond %{QUERY_STRING}   !(pimcore_editmode=true|pimcore_preview|pimcore_version)
RewriteCond %{DOCUMENT_ROOT}/var/tmp/pages%{REQUEST_URI}.html -f
RewriteRule ^(.*)$ /var/tmp/pages%{REQUEST_URI}.html [PT,L]

# cache-buster rule for scripts & stylesheets embedded using view helpers
RewriteRule ^cache-buster\-[\d]+/(.*) $1 [PT,L]

# If the requested filename exists, simply serve it.
# We only want to let Apache serve files and not directories.
RewriteCond %{REQUEST_FILENAME} -f
RewriteRule ^ - [L]

# Rewrite all other queries to the front controller.
RewriteRule ^ %{ENV:BASE}/index.php [L]




##########################################
### OPTIONAL PERFORMANCE OPTIMIZATIONS ###
##########################################

<IfModule mod_deflate.c>
    # Force compression for mangled headers.
    # http://developer.yahoo.com/blogs/ydn/posts/2010/12/pushing-beyond-gzipping
    <IfModule mod_setenvif.c>
        <IfModule mod_headers.c>
            SetEnvIfNoCase ^(Accept-EncodXng|X-cept-Encoding|X{15}|~{15}|-{15})$ ^((gzip|deflate)\s*,?\s*)+|[X~-]{4,13}$ HAVE_Accept-Encoding
            RequestHeader append Accept-Encoding "gzip,deflate" env=HAVE_Accept-Encoding
        </IfModule>
    </IfModule>

    # Compress all output labeled with one of the following MIME-types
    # (for Apache versions below 2.3.7, you don't need to enable `mod_filter`
    #  and can remove the `<IfModule mod_filter.c>` and `</IfModule>` lines
    #  as `AddOutputFilterByType` is still in the core directives).
    <IfModule mod_filter.c>
        AddOutputFilterByType DEFLATE application/atom+xml application/javascript application/json \
            application/vnd.ms-fontobject application/x-font-ttf application/rss+xml \
            application/x-web-app-manifest+json application/xhtml+xml \
            application/xml font/opentype image/svg+xml image/x-icon \
            text/css text/html text/plain text/x-component text/xml text/javascript
    </IfModule>
</IfModule>

<IfModule mod_expires.c>
    ExpiresActive on
    ExpiresDefault "access plus 31536000 seconds"

    # specific overrides
    #ExpiresByType text/css "access plus 1 year"
</IfModule>

<IfModule pagespeed_module>
    # pimcore mod_pagespeed integration
    # pimcore automatically disables mod_pagespeed in the following situations: debug-mode on, /admin, preview, editmode, ...
    # if you want to disable pagespeed for specific actions in pimcore you can use $this->disableBrowserCache() in your action
    RewriteCond %{REQUEST_URI} ^/(mod_)?pagespeed_(statistics|message|console|beacon|admin|global_admin)
    RewriteRule . - [L]

    ModPagespeed Off
    AddOutputFilterByType MOD_PAGESPEED_OUTPUT_FILTER text/html
    ModPagespeedModifyCachingHeaders off
    ModPagespeedRewriteLevel PassThrough
    # low risk filters
    ModPagespeedEnableFilters remove_comments,recompress_images
    # low and moderate filters, recommended filters, but can cause problems
    ModPagespeedEnableFilters lazyload_images,extend_cache_images,inline_preview_images,sprite_images
    ModPagespeedEnableFilters combine_css,rewrite_css,move_css_to_head,flatten_css_imports,extend_cache_css,prioritize_critical_css
    ModPagespeedEnableFilters extend_cache_scripts,combine_javascript,canonicalize_javascript_libraries,rewrite_javascript
    # high risk
    #ModPagespeedEnableFilters defer_javascript,local_storage_cache
</IfModule>
```


## Virtual Hosts
Make sure `Allowoverride All` is set for the `DocumentRoot`, which enables `.htaccess` support. 

#### Example 
```apacheconf
<VirtualHost *:443>
        ServerName YOUPROJECT.local
        DocumentRoot  /var/www/public

        <FilesMatch \.php$>
            SetHandler "proxy:unix:/var/run/php/pimcore.sock|fcgi://localhost"
        </FilesMatch>

        <Directory /var/www/public>
                Options FollowSymLinks
                AllowOverride All
                Require all granted
        </Directory>

        SSLEngine on
        # NEEDS TO BE CHANGED
        SSLCertificateFile /etc/getssl/YOUPROJECT.local/YOUPROJECT.local.crt
        SSLCertificateKeyFile /etc/getssl/YOUPROJECT.local/YOUPROJECT.local.key
        SSLCertificateChainFile /etc/getssl/YOUPROJECT.local/chain.crt

        RewriteEngine On

        # THE FOLLOWING NEEDS TO BE THE VERY LAST REWRITE RULE IN THIS VHOST
        # this is needed to pass the auth header correctly - fastcgi environment
        RewriteRule ".*" "-" [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]

        ErrorLog ${APACHE_LOG_DIR}/YOUPROJECT.local_443_error.log
        CustomLog ${APACHE_LOG_DIR}/YOUPROJECT.local_443_access.log combined
</VirtualHost>
```
