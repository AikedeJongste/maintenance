# Capistrano::Maintenance

## Installation

Add this line to your application's Gemfile:

``` ruby
gem 'capistrano', '~> 3.11'
gem 'capistrano-maintenance', '~> 1.2', require: false
```

And then execute:

``` bash
$ bundle
```

Or install it yourself:

``` bash
$ gem install capistrano-maintenance
```

## Usage

Before using the maintenance tasks, you need to configure your webserver. How you do this depends on how your server is configured, but the following examples should help you on your way.

Application servers such as [Passenger](https://www.phusionpassenger.com) and [unicorn](http://unicorn.bogomips.org) requires you to set your public web directory to `current/public`. Both examples below are compatible with this setup.

Here is an example config for **nginx**. Note that this is just a part of the complete config, and will probably require modifications.

``` nginx
error_page 503 @503;

# Return a 503 error if the maintenance page exists.
if (-f /var/www/domain.com/shared/public/system/maintenance.html) {
  return 503;
}

location @503 {
  # Serve static assets if found.
  if (-f $request_filename) {
    break;
  }

  # Set root to the shared directory.
  root /var/www/domain.com/shared/public;
  rewrite ^(.*)$ /system/maintenance.html break;
}
```

And here is an example config for **Apache**. This will also need to be modified.

``` apache
# Enable rewriting
RewriteEngine On
# Create an alias to the maintenance page used as error document.
Alias "/error" "/var/www/domain.com/shared/public/system/"
ErrorDocument 503 /error/maintenance.html

# Return 503 error if the maintenance page exists.
RewriteCond /var/www/domain.com/shared/public/system/maintenance.html -f
RewriteRule !^/error/maintenance.html - [L,R=503]

# Redirect all non-static requests to unicorn (or similar).
# Will not redirect any requests if the maintenance page exists.
RewriteCond /var/www/domain.com/shared/public/system/maintenance.html !-f
RewriteCond %{DOCUMENT_ROOT}/%{REQUEST_FILENAME} !-f
RewriteRule ^/(.*)$ balancer://unicornserver%{REQUEST_URI} [P,QSA,L]
```

You can now require the gem in your `Capfile`:

```
require 'capistrano/maintenance'
```

### Enable task

Present a maintenance page to visitors. Disables your application's web interface
by writing a `"#{maintenance_basename}.html"` file to each web server. The
servers must be configured to detect the presence of this file, and if
it is present, always display it instead of performing the request.

By default, the maintenance page will just say the site is down for
"maintenance", and will be back "shortly", but you can customize the
page by specifying the REASON and UNTIL environment variables:

```
cap maintenance:enable REASON="hardware upgrade" UNTIL="12pm Central Time"
```

You can use a different template for the maintenance page by setting the
`:maintenance_template_path` variable in your deploy.rb file with an absolute path.

```
set :maintenance_template_path, File.expand_path("../../app/path/to/maintenance.html.erb", __FILE__)
```

The template file should either be a plaintext or an erb file. For example:

``` html
<!DOCTYPE html>
<html>
<head>
    <title>Maintenance</title>
    <style type="text/css">
    body {
        width: 400px;
        margin: 100px auto;
        font: 300 120% "OpenSans", "Helvetica Neue", "Helvetica", Arial, Verdana, sans-serif;
    }

    h1 {
        font-weight: 300;
    }
    </style>
</head>
<body>
    <h1>Maintenance</h1>

    <p>The system is down for <%= reason ? reason : "maintenance" %><br>
           as of <%= Time.now.strftime("%F %H:%M %Z") %>.</p>

    <p>It'll be back <%= deadline ? deadline : "shortly" %>.</p>
</body>
</html>
```

You can customize which folder the maintenance page template is rendered to by
setting the `:maintenance_dirname` variable. By default it renders to
`"#{shared_path}/public/system"`.

```
set :maintenance_dirname, -> { "#{current_path}/dist" }
```

By default, the maintenance page will be uploaded to all servers with the `web` role,
but if your application has different needs, you can customize this using the
`maintenance_roles` variable:

```
set :maintenance_roles, -> { roles([:web, :other_role]) }
```

Further customization will require that you write your own task.

### Disable task

``` bash
cap maintenance:disable
```

Makes the application web-accessible again. Removes the
`"#{maintenance_basename}.html"` page generated by maintenance:enable, which (if your
web servers are configured correctly) will make your application web-accessible again.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## Authors

The original `capistrano-maintenance` gem was created by [Thomas von Deyen](https://github.com/tvdeyen) and the implementation for Capistrano 3 in this repo was written by [Kir Shatrov](https://github.com/kirs).

As a Capistrano team, we thank Thomas for collaboration and providing the push access to the `capistrano-maintenance` gem.
