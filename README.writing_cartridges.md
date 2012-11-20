# How To Write An OpenShift Origin Cartridge 2.0 #

OpenShift cartridges provide the necessary command and control for
the functionality of software that is running user's applications.
OpenShift currently has many language cartridges JBoss, PHP, Ruby
(Rails) etc. as well as many DB cartridges such as Postgres, Mysql,
Mongo etc. Before writing your own cartridge you should search the
current list of [Red Hat](https://openshift.redhat.com) and [OpenShift
Community](https://openshift.redhat.com/community) provided cartriges.

## Cartridge Directory Structure ##

This is the minimal structure to which your cartridge is expected to
conform when written to disk. Failure to meet these expections will
cause your cartridge to not function when either installed or used on
the system. You may have additional directories or files.  They will
be ignored by the system. Note these files are copied into each gear
using your cartridge therefore you should be frugal in using the gear's
resources.

    `cartridge name`-`cartridge version`
     +- bin
     |  +- setup
     |  +- teardown
     |  +- runhook
     |  +- control
     +- versions
     |  +- `software version`
     |  |  +- bin
     |  |  +- build
     +- env
     +- metadata
     |  +- manifest.yml
     |  +- root_files.txt

To support multiple software versions with your cartridge,
you may create symlinks between the bin/control and the setup
versions/{cartridge&nbsp;version}/bin/control file. Or, you may choose
to use the bin/control file as a shim to call the correct versioned
control file.

## Cartridge Locking ##

Cartridges on disk within a gear may be either `locked` or `unlocked` at
a given time. When a cartridge is unlocked, a subset of files and directories
(specified by `root_files.txt`) will be writable by the gear user to enable
the cartridge scripts to perform their work.

Unlocking allows the cartridge scripts to have additional access to the gear's 
files and directories. Other scripts and hooks written by the end user will 
not be able to override decisions you make as the cartridge author.

The lock state is controlled by the OpenShift infrastructure, and cartridges
are locked and unlocked at various points in the cartridge lifecycle.

If you fail to provide a `metadata/root_files.txt` file or the file
is empty, your cartridge will remain always locked. For a very simple cartridges 
this may be sufficient.

**Note on security:** Cartridge file locking is not intended to be a
security measure. It is a mechanism to help prevent cartridge users from
inadvertently breaking their apps by modifying files reserved for use
by the cartridge.

### Lock configuration ###

The `metadata/root_files.txt` lists the files and directories, one per
line, that will be provided to the cartridge author with read/write
access while the cartridge is unlocked, but only read access to the end
user while the cartridge is locked. Any non-existant files that are
included in the list will be created when the cartridge is unlocked.
Any missing parent directories will be created as needed. The list
is anchored at the gear's home directory.  Entries ending in slash
is processed as a directory.  Entries ending in asterisk are a list
of files.  Entries ending in an other character are concidered files.
The system will not attempt to change files to directories or vice versa,
and your cartridge may fail to operate if files are miscatergorized and
you depend on the system to create them.

#### Lock configuration example

Here is a `root_files.txt` for a PHP cartridge:

    .pearrc
    php-5.3/bin/
    php-5.3/bin/*
    php-5.3/conf/*

Note in the above list the files in the `php-5.3/conf` directory are
unlocked but the directory itself is not.  Directories like `.node-gyp`
and `.npm` in nodejs are **NOT** candidates to be created in this
manner as they require the gear to have read and write access. These
directories would need to be created by the nodejs setup script which
is run while the gear is unlocked.

The following list of directories are reserved:

    .ssh
    .sandbox
    .tmp
    app-root
    git

## Cartridge Scripts ##

How you implement the cartridge scripts in the `bin` directory is up to you as
the author. For easily configured software where your cartridge is just
installing one version, these scripts may include all the necessary
code. For complex configurations or multi-version support, you may
choose to write these scripts as shim code to setup the neccessary
environment before calling additional scripts you write. Or, you may
choose to create symlinks from these names to a name of your choosing.
Your API is the scripts and their associated actions.

A cartridge must implement the following scripts:

* `setup`: Description
* `teardown`: Description
* `runhook`: Description
* `control`: Description


## bin/setup

##### Usage

`setup [--version=<version>]`

##### Options

* `--version=<version>`: Selects which version of cartridge to install. If no version is provided, 
the default from `mainfest.yml` will be installed.

##### Summary

The `setup` script is responsible for creating and/or configuring the files that
were copied from the cartridge repository into the gear's directory. The `manifest.yaml` 
file is used by the system to create the environment within which the setup command will run.

Lock context: `unlocked`

##### Broker signals

Your cartridge may provide a service that is consumed by multiple gears
in one application. The system provides the orchestration neccessary
for you to publish this service or services.

ADD_ENV_VAR...


## bin/teardown

##### Usage

`teardown`

##### Summary

The `teardown` script prepares the gear for the cartridge to be
removed. This is not called when the gear is destroyed.  The `teardown`
script is only run when a cartridge is to be removed from the gear.
The gear will continue to operate minus the functionality of this
cartridge.

Lock context: `unlocked`

* jwh: If all future gears are scaled, should teardown just always be called? *

##### Broker signals

Your cartridge may provide a service that is consumed by multiple gears
in one application. The system provides the orchestration neccessary
for you to publish this service or services.

RM_ENV_VAR...


## bin/runhook

##### Usage

`runhook <action> <uuid>`

##### Options

* `action`: which user operation the cartridge should perform
* `uuid`: OPENSHIFT_GEAR_UUID *jwh: Do we still need this? It should always in the environment...*

##### Summary

Hooks are called by the system to allow the end user to control aspects
of the cartridge or the software controlled by the cartridge.

The `runhook` script is usually a shim to ensure the environment
is correct for running your `action` code.  Hooks are called by the
system when it requires an action that is or may be dependent on unique
properties of your cartridge or the code your cartridge controls. This
script may be writted as a shim to provide access to a SCL-supported
language.

The hooks/scripts provided by the user are installed in
`$OPENSHIFT_HOME/app-root/runtime/repo/.openshift/action-hooks`.
Non-existanct or Non-executable scripts must be ignored.

Return an exit status of 0 for successfull or unsupported
operations. Non-zero when an error occurres. Text written on stdout may
or may not be reported to the user.  Text returned on stderr will be
logged, again it may or may not be reported to the user.

Actions:

 * jwh: not sure of list yet *

Lock context: `locked`


## bin/control

##### Usage

`control <action>`

##### Options

* `action`: which operation the cartridge should perform.

##### Summary

The `control` script allows the system or user to control the state of the cartridge.

The actions that must be supported:

   * `start` start the software your cartridge controls
   * `stop` stop the software your cartridge controls
   * `status` return an 0 exit status if your cartridge code is running.
      Additionally, you may return user directed information to stdout.
      Errors may be return on stderr with an non-zero exit status.
   * `reload` your cartridge and it's controlled code needs to re-read their
      configuration information. Depending on the software your cartridge is controlling
      this may equate to a restart.
   * `restart` stop current process and start a new one for the code your cartridge controls
   * `tidy` all unused resources should be released. It is at your discretion to
      determine what should be released. Remember on some systems resources may be
      very limited. For example, `git gc...` or `rm .../logs/log.[0-9]`

Lock context: `locked`


## bin/build

##### Usage

`build`

##### Summary

The `build` script is called during the `git push` to perform builds of the user's new code.

Lock context: `locked`


Environment Variables
---------------------

Environment variables are use to communicate setup information between
this cartridge and others, and to the system.  The cartridge controlled
variables are stored in the env directory and will be loaded after
system provided environment variables but before your code is called.
The system provided environment variables will be loaded and available
to be used for all cartridge entry points.

*jwh: ruby 1.9 makes providing environments very easy. We should exploit that.*

### System Provided Variables (Read Only) ###
 * `HISTFILE` bash history file
 * `OPENSHIFT_APP_DNS` the application's fully qualifed domain name that your cartridge is a part of
 * `OPENSHIFT_APP_NAME` the validated user assigned name for the application. Black list is system dependent. 
 * `OPENSHIFT_APP_UUID` the system assigned UUID for the application
 * `OPENSHIFT_DATA_DIR` the directory where your cartridge may store data
 * `OPENSHIFT_GEAR_DNS` the gear's fully qualifed domain name that your cartridge is a part of. May or may not be equal to
                        `OPENSHIFT_APP_DNS`
 * `OPENSHIFT_GEAR_NAME` the system assigned name for the gear. May or may not be equal to `OPENSHIFT_APP_NAME`
 * `OPENSHIFT_GEAR_UUID` the system assigned UUID for the gear
 * `OPENSHIFT_HOMEDIR` the system assigned directory from the gear
 * `OPENSHIFT_INTERNAL_IP` the private IP address for this gear *jwh: may go away*
 * `OPENSHIFT_INTERNAL_PORT` the private PORT for this gear *jwh: may go away*
 * `OPENSHIFT_REPO_DIR` the directory where the software your cartrige controls is stored
 * `OPENSHIFT_TMP_DIR` the directory where your cartridge may store temporary data

### Examples of Cartridge Provided Variables ###

*jwh: Is this true? or should the manifest call these out and the broker do the work?*

 * `OPENSHIFT_MYSQL_DB_HOST`
 * `OPENSHIFT_MYSQL_DB_LOG_DIR`
 * `OPENSHIFT_MYSQL_DB_PASSWORD`
 * `OPENSHIFT_MYSQL_DB_PORT`
 * `OPENSHIFT_MYSQL_DB_SOCKET`
 * `OPENSHIFT_MYSQL_DB_URL`
 * `OPENSHIFT_MYSQL_DB_USERNAME`
 * `OPENSHIFT_PHP_IP`
 * `OPENSHIFT_PHP_LOG_DIR`
 * `OPENSHIFT_PHP_PORT`

*jwh: now these are very cartridge dependent*

 * JENKINS_URL
 * JENKINS_USERNAME
 * JENKINS_PASSWORD

Your environment variables should be prefixed with
`OPENSHIFT_{cartridge&nbsp;name}_` to prevent overwriting other cartridge
variables in the process environment space.  The software you are
controlling may require environment variables of it's own, for example:
`JENKINS_URL`. Those you would add to your `env` directory or include
in shim code in your `bin` scripts.

Cartridge provided environment variables are not validated by the
system. Your cartridge may fail to function if you write invalid data
to these files. For the `JENKINS_URL` you would be expected to write
the file `env/JENKINS_URL` with the following content:

`export JENKINS_URL='http://jenkins-domain.rhcloud.com/'`

You may assume the
`PATH=$OPENSHIFT_HOMEDIR/{cartridge&nbps;name}-{cartridge&nbsp;version}/bin:/bin:/usr/bin/`
when your code is executed. Any additional directories your code requires
will need to be set in your shim code.

To install any php cartridge:

     # cp -ad ./php-5.3 ~UUID/                 - Run as root
     # stickshift/unlock.rb UUID php-5.3       - Run as root
     $ ~/php-5.3/setup --version php-5.3       - Bulk of work, run as user, from ~UUID
     # stickshift/lock.rb UUID php-5.3         - Run as root
     $ ~/php-5.3/control start                 - Run as user

To remove a php cartridge:

     $ ~/php-5.3/control stop                  - Run as user
     # stickshift/unlock.rb UUID php-5.3       - Run as root
     $ ~/php-5.3/teardown                      - run as user, from ~UUID
     # stickshift/lock.rb UUID php-5.3         - Run as root

