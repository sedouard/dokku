# Deploying to Dokku

## Deploy tutorial

Once dokku has been configured with at least one user, applications can be deployed via a `git push` command. To quickly see dokku deployment in action, you can use the heroku Ruby on Rails example app.

```shell
git clone git@github.com:heroku/ruby-rails-sample.git
```

### Create the app

Create the application on the dokku host. You will need to ssh onto the host to run this command.

```shell
dokku apps:create ruby-rails-sample
```

### Create the backing services

When you create a new app, Dokku by default *does not* provide any datastores such as mysql or postgres. You will need to install plugins to handle that, but fortunately [dokku has official plugins](/dokku/plugins/#official-plugins-beta) for common datastores. Our sample app requires a Postgres service:

```shell
# install the postgres plugin
# plugin installation requires root, hence the user change
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git

# create a postgres service with the name rails-database
dokku postgres:create rails-database
```

> Each services may take a few moments to create.

### Linking backing services to applications

Once the service creation is complete, set the `POSTGRES_URL` environment variable by linking the service.

```shell
# each official datastore offers a `link` method to link a service to any application
dokku postgres:link rails-database ruby-rails-sample
```

> You can link a single service to multiple applications or use one service per application.

### Deploy the app

Now you can deploy the `ruby-rails-sample` app to your Dokku server. All you have to do is add a remote to name the app. Applications are created on-the-fly on the dokku server.

```shell
git remote add dokku dokku@dokku.me:ruby-rails-sample
git push dokku master
```

You should see output similar to the following:

```
Counting objects: 231, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (162/162), done.
Writing objects: 100% (231/231), 36.96 KiB | 0 bytes/s, done.
Total 231 (delta 93), reused 147 (delta 53)
-----> Cleaning up...
-----> Building ruby-rails-sample from herokuish...
-----> Adding BUILD_ENV to build environment...
-----> Ruby app detected
-----> Compiling Ruby/Rails
-----> Using Ruby version: ruby-2.2.1
-----> Installing dependencies using 1.9.7
       Running: bundle install --without development:test --path vendor/bundle --binstubs vendor/bundle/bin -j4 --deployment
       Fetching gem metadata from https://rubygems.org/...........
       Fetching version metadata from https://rubygems.org/...
       Fetching dependency metadata from https://rubygems.org/..
       Using rake 10.4.2

...
```

When the deploy finishes, the application's url will be shown.

```shell
=====> Application deployed:
       http://ruby-rails-sample.dokku.me
```

Dokku supports deploying applications via [Heroku buildpacks](https://devcenter.heroku.com/articles/buildpacks) with [Herokuish](https://github.com/gliderlabs/herokuish#buildpacks) or using a project's [dockerfile](https://docs.docker.com/reference/builder/).

### Removing a deployed app

You can also remove an application from your dokku installation. This will unlink all linked services and destroy any config related to the application. Note that linked services will retain their data for later use (or removal).

```shell
# replace APP with the name of your application
dokku apps:destroy APP
```

This will prompt you to verify the application's name before destroying it. You may also use the `--force` flag to circumvent this verification process:

```shell
# replace APP with the name of your application
dokku --force apps:destroy APP
```

### Adding deploy users

While it is possible to use password-based authorization to push to dokku, it is preferable to use key-based authentication for security. You can add your public key to the dokku user's `authorized_keys` file with the following command:

```shell
# replace dokku.me with your domain name or the host's IP
# replace root with your server's root user
cat ~/.ssh/id_rsa.pub | ssh root@dokku.com "sudo sshcommand acl-add dokku [description]"
```

### Deploying non-master branch

Dokku only supports deploying from its master branch, so if you'd like to deploy a different local branch use: ```git push dokku <local branch>:master```

You can also support pushing multiple branches using the [receive-branch](/dokku/development/plugin-triggers/#receive-branch) plugin trigger in a custom plugin.

### Deploying with private git submodules

Dokku uses git locally (i.e. not a docker image) to build its own copy of your app repo, including submodules. This is done as the `dokku` user. Therefore, in order to deploy private git submodules, you'll need to drop your deploy key in `/home/dokku/.ssh` and potentially add github.com (or your VCS host key) into `/home/dokku/.ssh/known_hosts`. The following test should help confirm you've done it correctly.

```shell
su - dokku
ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts
ssh -T git@github.com
```

Note that if the buildpack or dockerfile build process require ssh key access for other reasons, the above may not always apply.

### Specifying a custom buildpack

In certain cases you may want to specify a custom buildpack. While dokku uses herokuish to support all the [official heroku buildpacks](https://github.com/gliderlabs/herokuish#buildpacks), it is possible that the buildpack detection does not work well for your application. As well, you may wish to use a custom buildpack to handle specific application logic.

To use a specific buildpack, you can run the following dokku command:

```shell
# replace APP with the name of your application
# replace REPOSITORY_URL with your buildpack's url
dokku config:set APP BUILDPACK_URL=REPOSITORY_URL
```

Please check the documentation for your particular build pack as you may need to include configuration files (such as a Procfile) in your project root.

## Deploying to subdomains

The name of remote repository is used as the name of application to be deployed, as for example above:

    $ git remote add dokku dokku@dokku.me:ruby-rails-sample
    $ git push dokku master

Is deployed to,

    remote: -----> Application deployed:
    remote:        http://ruby-rails-sample.dokku.me

You can also specify fully qualified names, say `app.dokku.me`, as

    $ git remote add dokku dokku@dokku.me:app.dokku.me
    $ git push dokku master

So, after deployment the application will be available at,

    remote: -----> Application deployed:
    remote:        http://app.dokku.me

This is in particular useful, then you want to deploy to root domain, as

    $ git remote add dokku dokku@dokku.me:dokku.me
    $ git push dokku master

    ... deployment ...

    remote: -----> Application deployed:
    remote:        http://dokku.me

## Dokku/Docker Container Management Compatibility

Dokku is, at it's core, a docker container manager. Thus, it does not necessarily play well with other out-of-band processes interacting with the docker daemon. One thing to note as in [issue #1220](https://github.com/progrium/dokku/issues/1220), dokku executes a cleanup function prior to every deployment. This function removes all exited containers and all 'unattached' images.

## Default vhost

See the [nginx documentation](/dokku/nginx/#default-site).

## Dockerfile deployment

See the [dockerfile documentation](/dokku/deployment/dockerfiles/).

## Zero downtime deploy

See the [zero-downtime deploy documentation](/dokku/checks-examples/).

## Image tagging

See the [image tagging documentation](/dokku/deployment/images/).
