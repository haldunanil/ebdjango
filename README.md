# Installing Django on AWS Elastic Beanstalk with PostgreSQL and HTTPS
Recently, while attempting to set up an Elastic Beanstalk (EB) application on Amazon Web Services (AWS), I was frustrated to find that there are many disparate tutorials floating online that address the various parts of the set up process. The problem, however, is that when a particular blog post or walkthrough or whatever references another article, the installations and small underlying configurations tend to be slightly different. 

You can imagine that this is kind of frustrating, especially when certain parts of the setup could be lumped together, making the whole process a lot more effortless. This repo is an attempt to do just that (and a way for me to be able to get started on future projects quickly!). As a result, it starts from the beginning (literally, from installing Python on your local computer) and takes you through the entire journey of getting an Elastic Beanstalk up and running in record time with minimal tab switching. This guide is written for use on macOS Sierra, but many parts should be easily replicated on other systems.

Below are the packages and tools I've used throughout this walkthrough. In most cases, you should be able to do these tasks if you're on different versions, but I make no guarantees that the steps below will work with different versions of these tools and packages. Here they are: 
- Anaconda 4.3.0 running Python 3.6.0 (though you should also be able to follow along with virtualenv)
- Homebrew
- PostgreSQL (installed through Homebrew)
- Postgres.app (free online, link below)
- pip 9.0.1
- django 1.10.6
- psycopg2 2.7
- django-storages 1.5.2
- boto 2.46.1
- awsebcli 3.10.0

Without any further ado, let's get started!

## Step 1: Getting Anaconda on your computer

First things first: we need to get a working version of Python installed on our computer. This tutorial assumes you're using Anaconda to create your virtual environments, but you're more than welcome to skip the Anaconda installation and simply use virtualenv instead. If you want to do that, just install [Python 3.6](https://www.python.org/downloads/release/python-360/) and run `pip install virtualenv awsebcli` in terminal and skip to Step 2. 

1. Download and install anaconda for Python3 from https://www.continuum.io/downloads
2. Fire up the terminal and run the following code to update your Anaconda installation (if necessary), create a new virtual environment, and install the AWS EB command line interface to allow us to interact with EB through terminal. We'll install `awsebcli` in the main Anaconda installation rather than inside the virtual environment so we can make deployments without needing to go into a virtual env in the future. If you want to keep everything inside the virtual environment, then run `source activate ebenv` before you pip install awsebcli.
```shell
$ conda update conda
$ conda create -n ebenv python=3.6
$ pip install awsebcli
```

You may want to test out your installation to make sure it works; just type `source activate ebenv` in terminal and your terminal should change from
```shell
$ My-MacBook-Pro: ~myusername$
```

to:

```shell
(ebenv) $ My-MacBook-Pro: ~myusername$
```

If that happened, then the virtual environment was configured successfully.

## Step 2: Installing Homebrew + PostgreSQL

There are multiple ways of getting PostgreSQL onto your computer, but I found that the best way to install it and make sure that all the symlinks are in place is through Homebrew. The installer from PostgreSQL's own website was quite frustrating as I spent a while trying to get `psql` to do anything. Eventually, I gave up and just went with Homebrew instead. Conveniently, that's also where you'll get your git installation for version control, so it's really a win-win. Plus, I love Homebrew's little beer emojis as it "brews" your apps. :beers:

1. If it's not open already, fire up the terminal and run the following:
```shell
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
$ brew install postgresql
$ brew install git
```
2. Download and install [Postgres.app](https://postgresapp.com/). This will help you easily run a local PostgreSQL server. 
3. Launch the Postgres.app.
4. Check to make sure that the server port is `5432` from Server Settings.
5. Initialize the server by pressing "Start".
6. Now, go back to terminal. We're going to run the following, which will let you:
  1. Open a postgres command line interface 
  2. List all of the the databases currently available
  3. Create a new database called `maindb` that we'll use throught the rest of this tutorial
  4. Ensure that the right users exist in the database

```shell
$ psql
$ \list
$ CREATE DATABASE maindb ENCODING=utf8;
$ SELECT * FROM pg_user;
```

7. If the user `postgres` exists, then run `ALTER USER postgres PASSWORD 'postgres';`
8. If the user `postgres` does not exist, then run `CREATE USER postgres PASSWORD 'postgres';`
9. Quit the postgres cli with `\q` and go back to terminal.
```
Hopefully, you got no errors during that entire process. If that's the case, let's move on to create a virtual environment and begin installing the tools we'll need to make this happen!

## Step 3: Initializing virtualenv and creating a local Django installation

Now, let's start creating our django server! First things first: we want to make sure we have all the right packages in place. Run the following in terminal:
```shell
$ source activate ebenv
$ pip install django, django-storages, psycopg2, boto
$ django-admin startproject ebdjango
$ cd ebdjango
$ nano ebdjango/settings.py
```

Now, your terminal should be in editor mode with `settings.py` in place. Alternatively, feel free to open up `settings.py` with your favorite text editor (shoutout to [Sublime](sublimetext.com)). Once the file is open, continue following these steps. Replace `DATABASES` with the following snippet. You'll notice that the part in the first `if` statement refers to `os.environ`. This will let us link up to AWS' Relational Database Service (RDS), which we'll use to run our databases off of. 
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'maindb',
        'USER': 'postgres',
        'PASSWORD': 'postgres',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}   
```

eplace `ALLOWED_HOSTS` with the following snippet, changing `.example.com` with your own domain, if applicable.
```python
ALLOWED_HOSTS = ['.elasticbeanstalk.com',
                 'localhost',
                 '127.0.0.1',
                 '.example.com']
```                

This will let you run the website off AWS, locally, and through your domain without getting any annoying errors. Lastly:
1. Add `storages` to `INSTALLED_APPS`
2. Update timezone with your timezone, such that you get something like `TIME_ZONE = 'America/New_York'`
3. Change We're done with `settings.py` for now, time to save and close (Ctrl+X, Y, Enter). If you were using a text editor, go back to terminal and run the following:
```shell
$ mkdir .ebextensions
$ nano .ebextensions/01_packages.config
```

This lets us create a directory where we'll tell EB to install certain packages to your Elastic Compute Cloud (EC2) installations. Time for a quick sidebar for those who are new to Elastic Beanstalk or to AWS in general about the basic philosophy of EB. In a traditional EC2 server, you have individual instances running your website. These instances need to be maintained individually and are limited in how automatically they can be scaled should, say, your website suddenly experience a significant jump in traffic. If you're like me, you dread the idea of having to closely monitor your EC2 instances and worry about whether you're using a sufficient amount of resources, because you want to focus on the product itself. AWS Elastic Beanstalk is Amazin's wonderful solution to this problem: rather than set up individual EC2 instances, RDS databases, and S3 buckets, EB gives you a convenient way to set them all up in one go. 

An important difference here, in practice, is how you configure instances. For this tutorial, you need to make sure PostgreSQL is installed on your EC2 instances. In a traditional EC2 setup, you'd just SSH into the EC2 instance and use something like `yum install postgresql95`. This doesn't work so well with EB, as when the beanstalk auto-scales your instances up or down, all changes made to individual instances get destroyed, so you'd have to SSH into every new instance and reinstall the dependencies lest your website stop working, over and over and over again. Not fun. Instead, you use `.config` files that are placed into the folder we just created, `.ebextensions` and instruct the EB to make sure that all new EC2 instances have the pre-requisite tools and apps installed as they are deployed. Awesome, right? :bowtie:

To do this, copy the following code, go back to your terminal, and paste it into 01_packages.config:
```shell
packages:
  yum:
    git: []
    postgresql95: []
    postgresql95-devel: []
    postgresql95-server: []
    postgresql95-contrib: []
    postgresql95-docs: []
```

Save and close. Now, type:
```shell
$ nano .ebextensions/django.config
```

Into this new config file, paste the following, save and close:

```shell
option_settings:
  aws:elasticbeanstalk:application:environment:
    DJANGO_SETTINGS_MODULE: ebdjango.settings
    PYTHONPATH: $PYTHONPATH
  aws:elasticbeanstalk:container:python:
    WSGIPath: ebdjango/wsgi.py
```

In the above, we instruct the server to correctly use the Django configuration. To make sure that the the right python packages are installed onto the EC2 instances, we also want to make sure that we store a list of the required packages. To do so, enter:
```shell
$ pip freeze > requirements.txt
```

To make your life a little easier, it might be helpful to edit the `requirements.txt` and remove the versions to make sure that there are no errors during installation in case `pip` is unable to find the exact version of the package you're looking for. Obviously, this is potentially dangerous as different versions might introduce/remove features you're using, so exercise best judgment here (i.e. leave them if you're unsure).

## Step 3: Creating an AWS EB application, environemnts, and deployment

Now that we have a basic working version of the Django setup written, go back to the terminal and type the following:
```shell
$ eb init

Select a default region
1) us-east-1 : US East (N. Virginia)
2) us-west-1 : US West (N. California)
3) us-west-2 : US West (Oregon)
4) eu-west-1 : EU (Ireland)
5) eu-central-1 : EU (Frankfurt)
6) ap-south-1 : Asia Pacific (Mumbai)
7) ap-southeast-1 : Asia Pacific (Singapore)
8) ap-southeast-2 : Asia Pacific (Sydney)
9) ap-northeast-1 : Asia Pacific (Tokyo)
10) ap-northeast-2 : Asia Pacific (Seoul)
11) sa-east-1 : South America (Sao Paulo)
12) cn-north-1 : China (Beijing)
13) us-east-2 : US East (Ohio)
14) ca-central-1 : Canada (Central)
15) eu-west-2 : EU (London)
(default is 3): 
```

Select whatever is closest to you.

```shell
(aws-access-id): 
```

Here, you need to create an IAM account if you haven't already. Before you do anything on terminal, go to the IAM Management Console and [create a new user](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html). For simplicity, give the user `AWSElasticBeanstalkFullAccess` permissions. You may want to tweak this later on. Additionally, once you're all set up, you probably want to create groups. give permissions to those and include users within them to make user management easier. For info on groups, click [here](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups_create.html). Once you've created your user, copy the `Access key ID` and paste it in your terminal. Don't close that page, as you'll also need the `Secret access key` for the next step, which is:
```shell
(aws-secret-key):
```

After you enter your secret key, you'll get a new prompt:
```shell
Enter Application Name
(default is "django-postgresql-awsebcli"): 
```

You can just enter and continue for this exercise. In the future, just name your application something memorable. Next:

```shell
It appears you are using Python. Is this correct?
(Y/n): 
```

2 guesses: which is the right answer? SPOILER ALERT: type `Y` and continue.

```shell
Select a platform version.
1) Python 3.4
2) Python
3) Python 2.7
4) Python 3.4 (Preconfigured - Docker)
(default is 1): 
```

Press enter and continue (we want to be using Python 3.x).

```shell
Note: Elastic Beanstalk now supports AWS CodeCommit; a fully-managed source control service. To learn more, see Docs: https://aws.amazon.com/codecommit/
Do you wish to continue with CodeCommit? (y/N) (default is n): 
```

If you want to use AWS' version control service press y, otherwise just press enter and continue.

```shell
Do you want to set up SSH for your instances?
(Y/n): 
```

As mentioned earlier, SSH'ing into your EC2 instances in an EB setup is not a very good idea, so I select `n` and continue. Now that application is set up, we'll go ahead and get our environments going. Type the following:
```shell
$ eb create
Enter Environment Name
(default is django-postgresql-awsebcli-dev): 
```

Again, make it memorable. If you want to configure a staging/test/qa/etc. server in the future, just change the last bit from `-dev` and you're good to go. Press enter and now you'll see:
```shell
Enter DNS CNAME prefix
(default is django-postgresql-awsebcli-dev): 
```

If this is already installed, it won't work so change this up a little until the ebcli accepts it. Next up:
```shell
Select a load balancer type
1) classic
2) application
(default is 1): 
```

Press enter and keep going here and the EB CLI will begin to work its magic, creating all of your environments including EC2 instances, security groups, a load balancer, and more! This usually takes around 5 minutes. Once that's done, we want to deploy our newly minted Django app onto our EB application! Type the following:
```shell
$ eb open
```

![Django splash](https://github.com/haldunanil/ebdjango/blob/master/first-debug.png)

Congratulations, you just successfully deployed a Django site to AWS Elastic Beanstalk! Now, go celebrate, you champion.

## Step 4: Setting up RDS and a new Django app to let you login to admin

Now that the Django website is running, let's set up an RDS instance to connect our database! Go back to the terminal and type the following (or manually navigate to the EB console):
```shell
$ eb console
```

On the left hand side, you'll see a button that reads `Configuration`, press that and scroll to the bottom of the page that just popped up. Press `Create a new RDS database`. On the next page, enter values such that the page looks like this:

![RDS setup](https://github.com/haldunanil/ebdjango/blob/master/rds-setup.png)

Once you save the changes, EB will create the RDS. Afterward, go back to the terminal and type:

```shell
$ python manage.py migrate
$ python manage.py createsuperuser
Username (leave blank to use 'XXX'): admin
Email address: 
Password: 
Password (again): 
Superuser created successfully.
```

Now, we want to make sure that all the pieces of the admin interface works properly. To make sure that happens, add `STATIC_ROOT = 'static'` to the end of `settings.py`. Next, run the following:

```shell
$ python manage.py collectstatic
```

Now, let's set up our RDS server. Navigate to `settings.py` and replace the `DATABASE` section with:

```python
if 'RDS_DB_NAME' in os.environ:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': os.environ['RDS_DB_NAME'],
            'USER': os.environ['RDS_USERNAME'],
            'PASSWORD': os.environ['RDS_PASSWORD'],
            'HOST': os.environ['RDS_HOSTNAME'],
            'PORT': os.environ['RDS_PORT'],
        }
    }
else:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': 'maindb',
            'USER': 'postgres',
            'PASSWORD': 'postgres',
            'HOST': 'localhost',
            'PORT': '5432',
        }
    } 
```

Now, we need to create our first app. For the sake of simplicity, the only purpose of this app will be to create superusers during EB initialization. Of course, in your instance, you can use this app for other purposes as well. Go back to terminal and type these:

```shell
$ python manage.py startapp su
$ mkdir su/management
$ mkdir su/management/commands
$ nano su/management/__init__.py             # leave empty, save and close
$ nano su/management/commands/__init__.py    # leave empty, save and close
$ nano su/management/commands/createsu.py
```

Once you've started editing the `createsu.py`, paste the following:

```python
from django.core.management.base import BaseCommand
from django.contrib.auth.models import User

class Command(BaseCommand):

    def handle(self, *args, **options):
        if not User.objects.filter(username="admin").exists():
            User.objects.create_superuser("admin", "", "admin")
```

Reopen `settings.py` and enter `'su'` into `INSTALLED_APPS` to make sure that our Django server can use the superuser creator that we just wrote. Next, run the following:

```shell
$ nano .ebextensions/02_rds.config
```

Once you've begun editing, paste the following text in there:

```shell
container_commands:
  01_migrate:
    command: "python manage.py migrate"
    leader_only: true
  02_collectstatic:
    command: "python manage.py collectstatic --noinput"
  03_createsu:
    command: "python manage.py createsu"
    leader_only: true
```

We're at the homestretch! Run the following to deploy the updated version of our website to EB:

```shell
$ git add .
$ git commit -m "Migrate db, add staticfiles and createsu"
$ eb deploy
```

To check that everything works, let's now navigate to `XXX.elasticbeanstalk.com/admin` where `XXX` is the DNS name we've been using for our EB environment. Assuming everything went well, enter `admin` as your username and password to see the following page: 

![Admin page](https://github.com/haldunanil/ebdjango/blob/master/admin.png)

## Step 5: Setting up S3 and hooking it up to the Django config

Before proceeding, let's recall the core philosophy of AWS Elastic Beanstalk: EC2 instances can (and will) be auto-scaled depending on the volume of traffic coming to your website. As a result, we generally want to refrain from making any edits directly to any EC2 instance or store any static files there. As your website gets more and more pieces and content up, it'll be slower and more costly to attempt to serve everything from the EC2 instances. And what about other pieces, like profile images that may be uploaded by users? This is where S3 comes in: S3 provides a way to store static files and media independently of the EC2 instances, making sure that if/when EC2 instances are destroyed or spun up, all of your data continues to be available and also is served quickly on demand. Added bonus: when you configure CloudFront to work with your S3 instance, you can speed up the delivery of your static content even further!

Now, let's set this up. Start by adding the following to the bottom of your `settings.py` file:

```python
# S3 hookup

AWS_STORAGE_BUCKET_NAME = 'XXX'
AWS_ACCESS_KEY_ID = 'YYY'
AWS_SECRET_ACCESS_KEY = 'ZZZ'

# Tell django-storages that when coming up with the URL for an item in S3 storage, keep
# it simple - just use this domain plus the path. (If this isn't set, things get complicated).
# This controls how the `static` template tag from `staticfiles` gets expanded, if you're using it.
# We also use it in the next setting.
AWS_S3_CUSTOM_DOMAIN = '%s.s3.amazonaws.com' % AWS_STORAGE_BUCKET_NAME

# This is used by the `static` template tag from `static`, if you're using that. Or if anything else
# refers directly to STATIC_URL. So it's safest to always set it.
STATIC_URL = "https://%s/" % AWS_S3_CUSTOM_DOMAIN

# Tell the staticfiles app to use S3Boto storage when writing the collected static files (when
# you run `collectstatic`).
STATICFILES_STORAGE = 'storages.backends.s3boto.S3BotoStorage'
```

Now, we need to configure Cross-Origin Resource Sharing (CORS) on our S3 bucket to make sure that browsers visiting our website know to interact with resources (like images, CSS, JS, etc. files) coming from a different domain. To do so:
1. Go back to the S3 bucket
2. Open up `Properties`
3. Click `Add CORS Configuration`
4. Paste the following in there:

```html
<CORSConfiguration>
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>GET</AllowedMethod>
        <MaxAgeSeconds>3000</MaxAgeSeconds>
        <AllowedHeader>Authorization</AllowedHeader>
    </CORSRule>
</CORSConfiguration>
```

Note: it might already be there (it was in my case), in which case, just verify that it matches up.

Now, we've ensured that `static` files (the ones _we_ upload) are in S3. Next up, we want to ensure that `media` files (uploaded by _users_) also go on S3. We also want to make sure that they don't collide and start overwriting each other. There are two approaches to making this happen: 1) create separate directories in the S3 bucket for your `static` files and your `media` files or 2) create separate buckets for the two. To keep things together, we'll follow the first one. 

Now, let's go back to terminal, ensure that we're in the root folder `ebdjango` (the one with `manage.py` in it, NOT the one with `settings.py` in it). Let's do the following:

```shell
$ nano custom_storages.py
```

Into that file, paste the following:

```python
from django.conf import settings
from storages.backends.s3boto import S3BotoStorage

class StaticStorage(S3BotoStorage):
    location = settings.STATICFILES_LOCATION

class MediaStorage(S3BotoStorage):
    location = settings.MEDIAFILES_LOCATION
```

Save and close the file. As explained above, this file will route all static files into a folder called `/static/` and all the media files into a folder called `/media/`. We also need to make sure that our `settings.py` file recognizes this new file. To make it do so, we insert the following:

```python
STATICFILES_LOCATION = 'static'
STATICFILES_STORAGE = 'custom_storages.StaticStorage'
STATIC_URL = "https://%s/%s/" % (AWS_S3_CUSTOM_DOMAIN, STATICFILES_LOCATION)

MEDIAFILES_LOCATION = 'media'
DEFAULT_FILE_STORAGE = 'custom_storages.MediaStorage'
MEDIA_URL = "https://%s/%s/" % (AWS_S3_CUSTOM_DOMAIN, MEDIAFILES_LOCATION)
```

Let's deploy our website! Run the following:

```shell
$ eb deploy
$ eb open
```

Inspect individual elements to see that they're pulling CSS files and such from S3. If that's the case, you've done it!

## Step 6: Compressing elements to increase delivery speed

To make sure that everything happens quickly, let's set up gzip compression. To do so, run the following in terminal:

```shell
$ nano .ebextensions/03_apache.config
```

In that file, paste:

```shell
container_commands:
  01_setup_apache:
    command: "cp .ebextensions/enable_mod_deflate.conf /etc/httpd/conf.d/enable_mod_deflate.conf"
```

As you'll notice, we're referring to a file called `enable_mdo_deflate.conf`. Let's add that too:

```shell
$ nano .ebextensions/enable_mod_deflate.conf
```

Into _that_ file, insert:

```html
# mod_deflate configuration
<IfModule mod_deflate.c>
  # Restrict compression to these MIME types
  AddOutputFilterByType DEFLATE text/plain
  AddOutputFilterByType DEFLATE text/html
  AddOutputFilterByType DEFLATE application/xhtml+xml
  AddOutputFilterByType DEFLATE text/xml
  AddOutputFilterByType DEFLATE application/xml
  AddOutputFilterByType DEFLATE application/xml+rss
  AddOutputFilterByType DEFLATE application/x-javascript
  AddOutputFilterByType DEFLATE text/javascript
  AddOutputFilterByType DEFLATE text/css
  # Level of compression (Highest 9 - Lowest 1)
  DeflateCompressionLevel 9
  # Netscape 4.x has some problems.
  BrowserMatch ^Mozilla/4 gzip-only-text/html
  # Netscape 4.06-4.08 have some more problems
  BrowserMatch ^Mozilla/4\.0[678] no-gzip
  # MSIE masquerades as Netscape, but it is fine
  BrowserMatch \bMSI[E] !no-gzip !gzip-only-text/html
<IfModule mod_headers.c>
  # Make sure proxies don't deliver the wrong content
  Header append Vary User-Agent env=!dont-vary
</IfModule>
</IfModule>
```

Save and close. This should help speed up the delivery of your website!

## Step 7: Configuring the load balancer to route all traffic through HTTPS

Now that everything's up, we want to secure our website. Especially for websites where there's logins and such, using `HTTPS` is going to be an important aspect of making sure nobody's passwords get compromised. The great thing about AWS Elastic Beanstalk is that it enables a load balancer for you, which we'll use to route all traffic through `HTTPS`-- even if people use URLs that aren't secure. Let's get started! 

First, create an SSL certificate through AWS Certificate Manager (process is [here](http://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request.html)). Next, copy the ARN of the SSL certificate you created. Now, going back to terminal (remember to have it open in the root directory) and type:

```shell
$ nano .ebextensions/securelistener.config
```

Into the file, paste the following (making sure to replace `ARN_KEY_HERE` with the ARN key you copied moments ago):

```shell
option_settings:
  aws:elb:listener:443:
    SSLCertificateId: ARN_KEY_HERE
    ListenerProtocol: HTTPS
    InstancePort: 80    
```

Save and close-- you just configured your load balancer to accept HTTPS requests! Now, we add the Apache config that redirects all calls (whether it's `HTTP` or `HTTPS`) to HTTPS for every page. Start by creating an apache config file: 

```shell
$ nano .ebextensions/04_apache.config
```

Add the following:

```shell
files:
  "/etc/httpd/conf.d/ssl_rewrite.conf":
    mode: "000644"
    owner: root
    group: root
    content: |
      RewriteEngine On
      <If "-n '%{HTTP:X-Forwarded-Proto}' && %{HTTP:X-Forwarded-Proto} != 'https'">
      RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R,L]
      </If>
```              

In terminal, type:

```shell
$ eb deploy
$ eb open
```

Note that, unless you configure your DNS name with an ALIAS record to redirect to the EB application, the security note on the right-hand side will say `Not Secure`, since your security certificate does not include "elasticbeanstalk.com" domain(s). Let's fix that in the next section!

## Step 8: Make sure people can find you easily on the interwebz

We want to make sure that we can easily be found on the internet. You don't get to Google by typing `google-prod.elasticbeanstalk.com` so it doesn't make sense to do that with your website either. Assuming you've already purchased your own domain, here's how to assign proper forwarding for your website.

1. Go to AWS' `Route 53`
2. On the left-hand section, click on `Hosted zones`
3. Again, assuming you've already imported your main domain and configured your `NS` records (and if not, you can get started [here](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/welcome-domain-registration.html)), click on your domain
4. Now, click on `Create Record Set`
5. If you want to create a subdomain (like `test.example.com`, add its name in the `Name` section), if not skip to the next step
6. Leave `Type` as "A - IPv4 address"
7. Check "Yes" for `Alias`
8. For `Alias Target`, find "Elastic Beanstalk environments" and click the text underneath
9. Leave `Routing Policy` as "Simple"
10. Leave `Evaluate Target Health` as "No"
11. Click `Create`

Wait a few moments and navigate to the `Name` you just configured. You should see the home page for your website and your HTTPS tag should now say `Secured` with a green lock sign next to it. Congrats, your website is online! :confetti_ball: :

## Step 9: Goodbye and cleanup

With that, you probably want to clean up after yourself. Once you're done playing around with your website and basking in its glory, type:

```shell
$ eb terminate ebdjango-dev
```

The above will delete your environment, but NOT your application. For that (and to be sure):
1. Go to the Elastic Beanstalk Console
2. Make sure you're looking at the page titled `All Applications`
3. Click the `Actions` button on the right
4. Click `Delete Application`

AWS will take care of the rest. The application should disappear completely from your console within ~1 hour. That's it, you're done and all the additional resources created for you (EC2 instances, the RDS instance, etc. _excluding_ your S3 bucket) should also be deleted as part of the EB deletion process.

## Step 10: Quick start and closing thoughts

You're probably thinking to yourself: quick start? Now?! Yes. The reason why I put it down here is so you would follow along and learn how to deploy an EB application all by yourself rather than just copying off me. Give a man a fish, he eats for a day. _Teach_ a man how to fish...

In the future and for a quick start (or if you were lazy and just skipped straight down here), simply:
1. Fork the GitHub repo on this page
2. Open a console in the root directory
3. Run `$ eb init` and follow the steps in Step 3 above
4. Run `$ eb create` and keep following the steps (you should see a database connection error, don't worry about that)
5. Go to your Elastic Beanstalk console and follow the instructions at the beginning of Step 4 to create an RDS instance
6. Open up your `settings.py` file and edit your S3 location and permissions at the bottom; specifically the fields that are marked `XXXXX`, `YYYYY`, and `ZZZZZ`
7. Once everything's in place, deploy using `$ eb deploy`; you should get no errors!
8. Follow Step 8 to configure your DNS alias to forward to your Elastic Beanstalk instance

And you're done! See? Easy :)

If you have any unforeseen issues or errors while doing the above, please drop me a line and I'll try to fix whatever the issue may be. Happy web developing!
