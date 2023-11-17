# freebsd-puma 

An init script for running puma on FreeBSD.  
(Based on [freebsd-sidekiq](https://github.com/gamecreature/freebsd-sidekiq))

This script runs puma via the FreeBSD daemon tool. It should be compatible with the latest puma versions.
(Puma 5 dropped internal daemonizing support (<https://github.com/puma/puma/blob/master/5.0-Upgrade.md>))

Simply place the `puma` script in the `/usr/local/etc/rc.d` directory, modify it if necessary, 
and configure the application via variables in `/etc/rc.conf`

This has been tested on FreeBSD version from 12.1 - 13.2
It has been tested with ruby native, rbenv and rvm. 

## Make sure puma starts after external dependencies / accessories have been launched

The only thing that might need configuration is the `REQUIRE` line to specify the dependencies. 
By default it will make sure mysql-server, postgres or redis are started first

Others can be added to this list if the application requires them.
Note: These requires are only used if the services are running in the same (jailed) environment.


## Quick Setup

To get up and running quickly, adjust the `REQUIRE` line like above, and add edit `/etc/rc.conf`:

For Capistrano or Capistrano-like directory layouts:

```sh
puma_enable="YES"

# this is the path to the folder the  application is deployed (via for example Capistrano)
# (the parent directory of the `current` directory)
puma_directory="/u/application"
```

For Non-Capistrano-like layouts:

```sh
puma_enable="YES"
puma_command="/u/application/bin/puma"
puma_pidfile="/u/application/tmp/pids/puma.pid"
puma_config="/u/application/config/puma.rb"
puma_chdir="/u/application"
puma_user="deploy"
```


## Starting/Stopping/Restarting and Upgrading puma

Puma can be started like any other FreeBSD service:

```sh
service puma start
```

There's also a handy `show` command to check at the final puma configuration:

```sh
service puma show
```

To prepare shutdown, a quiet (prestop) command can be send. (kill TSTP)
Note: Tried to name it quiet, but that seems to be used by FreeBSD internally?

```sh
service puma stop
```

## `/etc/rc.conf` Details

### Using a Capistrano directory layout

The rc script does as much as possible. When using Capistrano, or a Capistrano-like directory structure, 
the only requirement is to specify the directory of the application (the parent directory of `current`):

```sh
puma_enable="YES"
puma_directory="/u/application"
```

This infers all sorts of information about the app.
(run `/usr/local/etc/rc.d/puma show` to see what the configuration is. 
 **Note** the names listed here are without the leading `puma_` prefix that need to be specified in `/etc/rc.conf`):

```text
command:        /u/app/current/bin/puma
command_args:   /u/app/current/config.ru
rackup:         /u/app/current/config.ru
pidfile:        /u/app/shared/tmp/pids/puma.pid
old_pidfile:    /u/app/shared/tmp/pids/puma.pid.oldbin
listen:
config:         /u/app/current/config/puma.rb
log:            /u/app/current/log/puma.log
init_config:    /u/app/current/.env
bundle_gemfile: /u/app/current/Gemfile
chdir:          /u/app/current
user:           user
nice:
env:            production
flags:          -e production -C /u/app/current/config/puma.rb
```

`start_command:`

```sh
su -l username -c "export BUNDLE_GEMFILE=/u/app/current/Gemfile && . /u/app/current/.env && cd /u/app/current && /usr/sbin/daemon -f -p /u/app/shared/tmp/pids/puma.pid -o /u/app/current/log/puma.log /u/app/current/bin/puma -e production -C /u/app/current/config/puma.rb  /u/app/current/config.ru "
```


Let's look at these settings one by one:

`command`: By default, it uses the `current/bin/puma` [bundler binstub][binstub] located in the project to ensure the gems are loaded. `command` comes from FreeBSD's `rc.subr` init system functions.
`command_args`: This is the standard FreeBSD's `rc.subr` variable that holds the arguments to the above `command`. Typically not required to set this.
`pidfile`: This is also part of FreeBSD's `rc.subr` system. This is where the built in functions will look for the pid of the process. By default, this rc script looks in the `current/tmp/pids/puma.pid` file.
`config`: This is the path to puma's config file where puma will find it's settings. By default this rc script looks for a file called `current/config/puma.rb`
`init_config`: This is a shell script file that is included in the environment before puma is executed. Include `export VAR=value` statements to pass environment variables into the app. By default, this init script looks for a file called `current/.env` and uses that. If that file doesn't exist, this rc script will skip this functionality (as seen in the above example).


This could be used in conjunction with [dotenv][dotenv] in development since dotenv accepts lines beginning with `export`

`bundle_gemfile`: This is the path to the `Gemfile` of the project. This rc script sets the `BUNDLE_GEMFILE` environment variable to this value. By default it looks to `current/Gemfile`. This is required so that puma uses the most current `Gemfile` (rather than the one in the specific deployment directory) when an upgrade is performed.
`chdir`: This is the directory we `cd` into before running puma. By default it's the currently deployed version of the application "`current/`"
`user`: This is the user that puma will be run as.
`nice`: The `nice` level to run puma at. Usually not changed 
`flags`: This is a variable defined by FreeBSD's `/etc/rc.subr` init system, and contains the flags passed to the command (puma in this case) when run. This variable is built up from the variables above, can be specified manually `puma_flags` in `/etc/rc.conf` to override them.
`start_command`: The full command that will be run when the service is started. 

Override any of these parameter in `/etc/rc.conf` by simply specifying the variables as below (pick and choose which to override).


### Using a custom directory layout

Using a custom layout is easy, just leave the `puma_directory` variable out of the `/etc/rc.conf` and specify all of 
the variables manually. Here's a list of those variables available.

```text
puma_command:      The path to the puma command
puma_command_args: The non-flag arguments passed to the above command.  Typically no changes are required.
puma_pidfile:      The path where puma will put its pid file
puma_config:       The path to the puma config file
puma_log:          The path to the puma log file
puma_chdir:        The path where this script will `cd` to before starting puma
puma_user:         The user to run puma as
puma_nice:         The `nice` level to run puma as. Leave blank to run un-niced
puma_env:          The RAILS_ENV (or RACK_ENV) to run the application in. (default: production)
puma_flags:        The flags passed in to puma when starting (not counting the puma_command_args specified above). Override this for complete control of how to start puma.
```

### Deploying multiple applications

This rc script can work with multiple applications. 
It works similarly to how postgresql rc script works on FreeBSD.

Profiles can be supplpied in `/etc/rc.conf` with the `puma_profiles` variable. 
This is a space separated list of application names.

Then each application can be customized by specifying variables in this form:

`puma_<application-name>_variable=VALUE`

Here's a simple example (The `_env` variable is skipeed for production since it's the default value)

```sh
puma_enable="YES"
puma_profiles="application_staging application_production"

puma_application_staging_enable="YES"
puma_application_staging_directory="/u/application_staging"
puma_application_staging_env="staging"

puma_application_production_enable="YES"
puma_application_production_directory="/u/application_production"
```

The example above assumes a capistrano like `directory` structure.
It is also possible to specify all of the variable's separately, for a fully custom setup.

### Customizing the script

To cusomtize the default, look at the `_setup_directory()` function to see the generated variables.
This is what's invoked when using a Capistrano-like directory layout by specifying `puma_directory` in `/etc/rc.conf`.

For different deployment adjust the default values to work with the system.

[freebsd-puma]: https://github.com/snake66/freebsd-puma
[dotenv]: https://github.com/bkeepers/dotenv
[binstub]: https://github.com/sstephenson/rbenv/wiki/Understanding-binstubs
[freebsd-unicorn]: https://github.com/caleb/freebsd-unicorn

