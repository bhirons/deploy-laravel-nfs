# deploy-laravel-nfs
Running notes on deploying a [Laravel](https://laravel.com/) 5.4+ site to [nearlyfreespeech.net](nearlyfreespeech.net) shared server host from a [github](github.com) repository.

## Differentiation
Existing guides document similar on more widely known shared hosting environments, foremost to [https://github.com/petehouston/laravel-deploy-on-shared-hosting](https://github.com/petehouston/laravel-deploy-on-shared-hosting) or probably a newer version someehere. I encountered variations specific to nearlyfreespeech.net.

Additional references are linked as needed throughout. 

# Requirements
Laravel [has requirements](https://laravel.com/docs/master/installation#server-requirements) that are important to this methods presentation.

I have found that the general purpose Apache/PHP server option available when creating a site on nearlyfreespeech.net is a sufficient server out-of-the-box for a vanilla Laravel instance.

To do this you need a funded account on nearlyfreespeech.net, and to create a site. If you have a domain, then that is nice, but you can create a site there without one. See their [excellent support options and doumentation here](https://members.nearlyfreespeech.net/buddhironsjr/support).

This guide includes deploying from github, so there is also an assumption that you are deploying from an existing repository. Although if you just want to install and run a fresh instance of Laravel for some reason, you can see how that is done here as well-- the difference being you install composer first, and then create a new laravel project.
 
# Start Doing it here
Installing a project named `laravelproject` 

> in the sample terminal commands below I always start with a `cd` command because I want to avoid any confusion about where things are happening. I am not a terminal guy from the start, unless you count me using a DOS prompt to start SpaceWar

## Create a site on nearlyfreespeech.net
Using the web admin page, create a new site, [instructions start here](https://members.nearlyfreespeech.net/faq?q=CreateSite#CreateSite)

After your site is created, go to the site admin pages, and scroll down to the Config section, make sure the **PHP version is set to 7.1**, otherwise there will be an error later. Click `edit` to change the PHP version as needed.

>if you run with PHP 5.x and Laravel 5.5 here you will get an error like _unexpected '?' in ..\vendor\laravel\framework\src\Illuminate\Foundation\helpers.php on line 233_ later. See the [Laracasts topic here](https://laracasts.com/discuss/channels/laravel/laravel-55-syntax-error-unexpected-in-vendorlaravelframeworksrcilluminatefoundationhelpersphp-on-line-233)

## Connect to your host
Have your account credentials handy as needed, and create a site, and then login to your [site using ssh](https://members.nearlyfreespeech.net/faq?q=SSH#SSH)

The shared host has a directory structure that includes these bits:

    /home
        /public
        /protected
        /...


The `/home/public` folder is the web root, and the `/home/protected` folder is where we install the Larevel app. On nearlyfreespeech, the `/home/protected` folder is meant for scripts or tools that support web applications and sites, but are not meant to be openly available to browsing. 

Laravel has its own public folder inside its own folder, and thus the difficulty of installing Laravel on your average shared host. The web server will expect to serve files from `/home/public`, but Laravel desires to serve the start page from `/home/public/laravelfolder/public`, so following the established approach, we will put the Laravel folder in `/home/protected/laravelfolder`, and utilize a symlink.

## First Deployment from github
To easily fetch from github, you should set up a deployment key. This will set up a trust relationship between your host and github so you can dispense with passwords or create deployment scripts later or similar.

### Generate your key
Once you have an open shell on your nearlyfreespeech host, you need to generate a new ssh key using the ssh-keygen tool, see this [doc from github](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/#generating-a-new-ssh-key)

You need to copy the public key from the output and use it as a deploy key on github. The `pbcopy` command is not available here, so just echo the public key out using the `cat` tool, and highlight copy the raw text output from your terminal.

   $ cat ~/.ssh/id_rsa.pub

### Install your key on Github
Then go to your repository page on Github, and click _Settings_ and then _Deploy Keys_ from the sidebar, and then the _Add deploy key_ and paste your clipboard there.

This sets up a key for only this repository, which is sufficient. Now you can issue git clone and fetch from your host, and the authentication will be handled for you using these keys.

### Clone from Github
Back to your terminal where you are connected to your host. From the `/home/protected` folder, you need to clone your repo here. Use `clone` here because it will setup the origin information, and create your new project folder. This means you can deploy updates from the same repo later without mucking about with archives etc.

    $ cd /home/protected
    $ git clone git@github.com:yourusername/laravelproject.git


## Configure the application on the host

These are all the housekeeping steps that have to be addressed on the first deployment.

### Install composer
Composer is required to handle dependencies for Laravel, but it is not installed by default. We can install it in the project folder, or if you are going to have a few sites on this server, install it in `/home/protected` so you don't have to have multiple copies.

Here we put it in the project folder, and follow the instructions from [the Composer docs](https://getcomposer.org/download/)

    $ cd /home/protected/laravelproject
    $ php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
    $ php -r "if (hash_file('SHA384', 'composer-setup.php') === '544e09ee996cdf60ece3804abc52599c22b1f40f4323403c44d44fdfdd586475ca9813a858088ffbc1f233e9b180f061') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
    $ php composer-setup.php
    $ php -r "unlink('composer-setup.php');"

If you like, you can rename the file to just `composer`, which I like to do so I can type less later.

    $ mv composer.phar composer

### Do the shared hosting workaround
This is online in a few places, most notably from Pete Houston's readme linked above. The notion is to create a symlink between the public folder in the `/home/protected/laravelproject/public` and `/home/public`. And then copy the relevant files into place once the link is established. We back things up just n case. 

    $ cd /home/protected/laravelproject
    $ mv public public_bak
    $ ln -s /home/public public
    $ cp -a public_bak/* public/
    $ cp public_bak/.htaccess public/

Now the paths in Laravel's index.php need to be updated. This is teh file that kicks of bootstrapping the Laravel application on a new request. So we need to change some include paths here and everything will just work throughout the lifetime of a request. So edit `index.php` and change the paths for the PHP require statement

    $ cd /home/public
    $ nano index.php

Change this line in the `Register The Autoloader` section

    require __DIR__.’/../bootstrap/autoload.php’;

to use the new paths

    require __DIR__.'/../protected/laravelproject/bootstrap/autoload.php';

And make a similar change in the `Turn on the lights` section.

    $app = require_once __DIR__.’/../bootstrap/app.php’;

becomes

    $app = require_once __DIR__.'/../protected/laravelproject/bootstrap/app.php';

Save and exit.

### Standard permission for the storage folder
The web server will have to be able to write cached views and logs, so set the permissions to allow that. This command resursively sets the write permission and ownsersip starting with the `storage` folder and for all children.

    $ cd /home/protected/laravelproject
    $ chmod -R o+w storage

### Install first-time dependencies
Now that everything is in place, we can run composer and update the dependencies, and configure our environment file.

> I don't recall off hand if the order here is important or not.

Create your `env` file

    $ cd /home/protected/laravelproject
    $ cp .env.example .env
    
Edit your `env` file, setting up the app stuff like url and app name and so on.

    $ cd /home/protected/laravelproject 
    $ nano .env 

Run composer, assuming a production environment.

    $ cd /home/protected/laravelproject
    $ php composer install --no-dev

Somewhere in here you need to set the application key as well

    $ cd /home/protected/laravelproject
    $ php artisan key:generate


## Maintenance Deployments
These are the steps to take on subsequent deployments.

Connect using `ssh` as noted above.

    $ cd /home/protected/laravelfolder
    $ git fetch --all
    $ git checkout --force "origin/master"

`origin/master` should be replaced with the tag you are using to deploy into production, or possibly your branch. Whichever the target happens to be based on your repo tactics. This technique detaches the HEAD on your working copy, so assumes that you are not going to edit anything here and push it back to your repo which is a best-practice.

#### The reason for the workaround
Since `public` was changed to a symlink above, and git stores the type flag for folders, it will overwrite the symlink setting public back to a folder. 

After fetching as above run through these steps to restore the symlink setup while preserving any previous edits in the files in the `/home/public/` folder. So we now have to take the step to re-do that, and preserve our modified index.php by backing it up to our private folder, and we don't have to copy htaccess this time. This could be scripted.

    $ cd /home/protected/laravelfolder
    $ cp /home/public/index.php /home/private/index.php
    $ rm -rf public_bak
    $ mv public public_bak
    $ ln -s /home/public public
    $ cp -a public_bak/* public/
    $ cp /home/private/index.php public/

#### Update dependencies

Run composer to install specified dependencies. You may have changed some dependencies in dev, and you want those on prod. You may have run `composer update` somewhere in dev, and tested it, and committed it, and your `composer.lock` stored in git reflects the dependencies you tested.

    $ cd /home/protected/laravelproject
    $ php composer install --no-dev

>if you are new to composer you will like to know this before you break a prod site.  See the [best explanation here](https://adamcod.es/2013/03/07/composer-install-vs-composer-update.html)


## Other Notes and Dependencies

### Database
nearlyfreespeech.net provides a MySQL database provider, actually implemented using the binary compatible MariaDB, but referenced in their documentaiton a MySQL anyway. Here is what you need to know.

#### MariaDB issue
For both 5.4 and 5.5 I had to add a fix for a key length size, since nearlyfreespeech is running MariaDB, and although it it all compatible and stuff, there are a few edge differences it turns out. If you get a key length error, you can edit `AppServiceProvider.php` to fix it. [See this topic for discussion](https://laravel-news.com/laravel-5-4-key-too-long-error).

Edit AppServiceProvider.php

    $ cd /home/protected/laravelproject/app/Providers
    $ nano AppServiceProvider.php

 And make sure it includes this:

    use Illuminate\Support\Facades\Schema;
    
    public function boot()
    {
        Schema::defaultStringLength(191);
    }


#### Create your MySQL process
nearlyfreesapeech.net lets you create a _MySQL process_ that can then contain multiple databases. The term _process_ here is equivalent to your separate db server often found in typical infrastructure setups, and the DSN name you create is what you will use for the `DB_HOST` config value. You can create a process using the web admin pages, and then using the connection info from the MySQL process web admin page, configure your Laravel site to connect. See the MySQL section of the FAQ. Specifically, [create a process](https://members.nearlyfreespeech.net/faq?q=SetupMySQL#SetupMySQL), [process vs. database](https://members.nearlyfreespeech.net/faq?q=MySQLDatabaseProcess#MySQLDatabaseProcess) and [create a database](https://members.nearlyfreespeech.net/faq?q=CreateDatabase#CreateDatabase).

PhpMyAdmin is available from a sidebar link in the admin web pages for creating a database before you try to use it with your Laravel site. The recommendation is that you create a user for each application for each of your application's database. Your application doesn't need to be able to do everything on all your databases, and the nearlyfreespeech directive is to never use your root access to a process for an application to do crud on a single DB. Follow this advice. Using PhpMyAdmin, create a database, select the database, and then create a user from the Privileges tab. Then you are going to edit this new user info into the `env` file. [PhpMyADmin docs are here](https://www.phpmyadmin.net/docs/).

    $ cd /home/protected/laravelproject
    $ nano .env

And then edit the DB info here.

    DB_CONNECTION=mysql
    DB_HOST=databaseprocessname.db
    DB_PORT=3306
    DB_DATABASE=databasenameyoucreated
    DB_USERNAME=databaseuseryourcreatedforthissite
    DB_PASSWORD=thepasswordfortheuser

After this is configured, you should be able to run migrations, and possibly any seeds as needed.



## Secure your site browsing
Setting up tls for your users to access your site securely using https is simple on this host.

#### Install the certificate
Once you are logged in with your terminal to your host, you can easily setup https using a script provided by nearlyfreespeech.net that takes care of [setting this up for free via LetsEncrypt as described here](https://members.nearlyfreespeech.net/faq?q=SSLCertificates#SSLCertificates). Or you can install your existing cert info if that is your preference.

    $ cd /home/public/
    $ tls-setup.sh

#### Force https
If you prefer to force Laravel to always use https even if a user types in a URL that is just http, you can do that [with a middleware](https://stackoverflow.com/questions/28402726/laravel-5-redirect-to-https), [a config setting](https://stackoverflow.com/questions/35827062/how-to-force-laravel-project-to-use-https-for-all-routes), and a [URL facade method around your routes](https://laracasts.com/discuss/channels/laravel/how-i-can-force-all-my-routes-to-be-https-not-http?page=1).

Also, make sure you set the `APP_URL` in your `env` file to be prefixed with https, so that when Laravel renders a url or resolves a route, it will use https by default.