 Deploy guide for AWS EC2
===============================
Hello, this is my simple guide for deploing your rails app 
There are many options to deploy your Rails application.
I will cover how to deploy a Rails application to Amazon Web Services (AWS) using Capistrano.


We will use the Puma + Nginx + PostgreSQL stack. 
Puma will be the application server, Nginx the reverse proxy, and PostgreSQL is the database server. 
Most of the steps remain same for both rubies, but I’ll highlight where they differ as well.

I also attach files with settings for Capistrano and Puma in this repo
 
 Configuring Puma and Capistrano
 --------------------------------------------
    gem 'puma'
    group :development do
      gem 'capistrano'
      gem 'capistrano3-puma'
      gem 'capistrano-rails', require: false
      gem 'capistrano-bundler', require: false
      gem 'capistrano-rvm'
    end
    
Install the gems via bundler:

    bundle install
    
It’s time to configure Capistrano:


       cap install STAGES=production
    
This comand will create configuration files for Capistrano at config/deploy.rb 
and config/deploy/production.rb. deploy.rb is the main configuration file
and production.rb contains environment specific settings, such as server IP, username, etc.
  
Add the following lines into the Capfile, 
found in the root of the application. The Capfile includes RVM, Rails, 
and Puma integration tasks when finished:

    require 'capistrano/bundler'
    require 'capistrano/rvm'
    require 'capistrano/rails/assets'
    require 'capistrano/rails/migrations'
    require 'capistrano/puma'
    
Edit deploy.rb:
     
    set :application, 'contactbook'
    set :repo_url, 'git@github.com:NikitaNaumenko/example.git' # Edit this to match your repository
    set :branch, :master
    set :deploy_to, '/home/deploy/contactbook'
    set :pty, true
    set :linked_files, %w{config/database.yml config/application.yml}
    set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system public/uploads}
    set :keep_releases, 5
    set :rvm_type, :user
    set :rvm_ruby_version, 'ruby-2.3.0' # Edit this if you are using MRI Ruby
    
    set :puma_rackup, -> { File.join(current_path, 'config.ru') }
    set :puma_state, "#{shared_path}/tmp/pids/puma.state"
    set :puma_pid, "#{shared_path}/tmp/pids/puma.pid"
    set :puma_bind, "unix://#{shared_path}/tmp/sockets/puma.sock"
    set :puma_conf, "#{shared_path}/puma.rb"
    set :puma_access_log, "#{shared_path}/log/puma_error.log"
    set :puma_error_log, "#{shared_path}/log/puma_access.log"
    set :puma_role, :app
    set :puma_env, fetch(:rack_env, fetch(:rails_env, 'production'))
    set :puma_threads, [0, 8]
    set :puma_workers, 0
    set :puma_worker_timeout, nil
    set :puma_init_active_record, true
    set :puma_preload_app, false

When you create server  Add Rule ‘HTTP’ from ‘Type’. This is required to make nginx server accessible from the Internet

Setup the server
----------------------------------
Connect to server :
    
    ssh -i "aws.pem" ubuntu@53.123.23.54
    
Update the existing packages first:

    sudo apt-get update && sudo apt-get -y upgrade

Create a user named deploy for deploying the application code:

    sudo useradd -d /home/deploy -m deploy

This will create the user deploy along with its home directory.
The application will be deployed into this directory. Set the password for deploy user:

    sudo passwd deploy

This password will be required by RVM for the Ruby installation. 
Also, add the deploy user to sudoers as well. Run sudo visudo and paste the following into the file:

    deploy ALL=(ALL:ALL) ALL

We will generate a key pair for that user now:

    su - deploy
    ssh-keygen
    
Do not set a passphrase for the key as it will be used as deploy key.

    cat .ssh/id_rsa.pub

Capistrano will connect to the server via ssh for deployment as the deploy account. 
Since AWS allows public key authentication only, copy the public key from your local machine to the deploy user account on the EC2 instance. 
The public key is your default ~/.ssh/id_rsa.pub key, in most cases. On the server:

    nano .ssh/authorized_keys

And paste your key into the file.
 
Install Git:

    sudo apt-get install git

#####Installing Nginx
Install nginx

    sudo apt-get install nginx
    
Configure the default site as our requirement. Open the site config file:

    sudo nano /etc/nginx/sites-available/default

Copy content and paste into the file:
    
    upstream app {
      # Path to Puma SOCK file, as defined previously
      server unix:/home/deploy/your-project/shared/tmp/sockets/puma.sock fail_timeout=0;
    }
    
    server {
      listen 80;
      server_name localhost;
    
      root /home/deploy/your-project/public;
    
      try_files $uri/index.html $uri @app;
    
      location / {
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_pass http://app;
      }
    
      location ~ ^/(assets|fonts|system)/|favicon.ico|robots.txt {
        gzip_static on;
        expires max;
        add_header Cache-Control public;
      }
    
      error_page 500 502 503 504 /500.html;
      client_max_body_size 4G;
      keepalive_timeout 10;
    }

Save the file and exit. We have configured nginx as a reverse proxy to redirect HTTP requests to the Puma application server through a UNIX socket. We will not restart nginx just yet, as the application is ready

#####Installing PostgreSQL

    sudo apt-get install postgresql postgresql-contrib libpq-dev

After installted, create a production database and its user:

    sudo -u postgres createuser -s user-name
    
Set the user’s password from psql console:

    sudo -u postgres psql

After logging into the console, change the password:

    postgres=# \password user-name

Enter your new password and confirm it.

Create a database for our application:

    sudo -u postgres createdb -O user-name your-project-name_production

#####Install node.js

    sudo apt-get install nodejs
    
#####Installing RVM & Ruby

    su - deploy
    gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
    \curl -sSL https://get.rvm.io | bash -s stable
    source ~/.rvm/scripts/rvm
    type rvm | head -n 1
    
    
You need   ***rvm is a function***

For using MRI Ruby:
    
    rvm install ruby-your-ruby-version
    
Install bundler:

    gem install bundler --no-ri --no-rdoc

Create the directories and files required by Capistrano. 
We will create the database.yml and application.yml files to store the database settings and other environment specific data:

    mkdir your-project-name
    mkdir -p your-project-name/shared/config
    nano your-project-name/shared/config/database.yml

Paste the following in database.yml:
    
    production:
      adapter: postgresql
      encoding: unicode
      database: your-project-name_production
      username: your-project-user-name
      password: your-project-user-name-password
      host: localhost
      port: 5432

Configure secrets.yml
    
    nano your-project-name/shared/config/secrets.yml

Paste the following in secrets.yml:

    production:
          secret_key_base: your-secret-key
          
Change the secret to a new secret using the rake secret command.

We’re almost done with the server. Go back to your local machine to start deployment with Capistrano. Edit the config/deploy/production.rb to set the server IP.

    server '53.123.23.54', user: 'deploy', roles: %w{web app db}

Now let’s start the deployment using Capistrano:

    cap production deploy

Restart nginx:

    sudo service nginx restart

Open up the browser:

    ubuntu@ec2-53-123-23-54.compute-1.amazonaws.com

