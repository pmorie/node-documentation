# How To Write An OpenShift Origin Cartridge 2.0 #

OpenShift cartridges provide the necessary command and control for
the functionality of software that is running user's applications.
OpenShift currently has many language cartridges JBoss, PHP, Ruby
(Rails) etc. as well as many DB cartridges such as Postgres, Mysql,
Mongo etc. Before writing your own cartridge you should search the
current list of [Red Hat](https://openshift.redhat.com) and [OpenShift
Community](https://openshift.redhat.com/community) provided cartridges.

## Cartridge Directory Structure ##

This is the minimal structure to which your cartridge is expected to
conform when written to disk. Failure to meet these expectations will
cause your cartridge to not function when either installed or used on
OpenShift. You may have additional directories or files.  They will
be ignored by OpenShift. Note these files are copied into each gear
using your cartridge therefore you should be frugal in using the gear's
resources.

    `cartridge name`-`cartridge version`
     +- bin
     |  +- setup
     |  +- teardown
     |  +- runhook
     |  +- control
     |  +- build
     +- versions
     |  +- `software version`
     |  |  +- bin
     |  |     +- build
     +- env
     +- metadata
     |  +- manifest.yml
     |  +- root_files.txt

To support multiple software versions with your cartridge,
you may create symlinks between the bin/control and the setup
versions/{cartridge&nbsp;version}/bin/control file. Or, you may choose
to use the bin/control file as a shim to call the correct versioned
control file.

## Cartridge Metadata ##

The `manifest.yaml` file is used by OpenShift to determine what features
your cartridge requires and in turn publishes. OpenShift also uses fields
in the `manifest.yml` to determine what data to present to the end user
about your cartridge.

An example `manifest.yml` file:

```yaml
Name: diy-0.1
Display-Name: diy v1.0.0 (noarch)
Description: "Experimental cartridge providing a way to try unsupported languages, frameworks, and middleware on OpenShift"
Version: 1.0.0
License: "ASL 2.0"
License-Url: http://www.apache.org/licenses/LICENSE-2.0.txt
Vendor:
Categories:
  - cartridge
  - web-framework
Website:
Help-Topics:
  "Getting Started": https://www.openshift.com/community/videos/getting-started-with-diy-applications-on-openshift
Cart-Data:
  - Key: OPENSHIFT_...
    Type: environment
    Description: "How environment variable should be used"
Suggests:

Provides:
  - "diy-0.1"
Requires:
Conflicts:
Native-Requires:
Architecture: noarch
Publishes:
  get-doc-root:
    Type: "FILESYSTEM:doc-root"
  publish-http-url:
    Type: "NET_TCP:httpd-proxy-info"
  publish-gear-endpoint:
    Type: "NET_TCP:gear-endpoint-info"
Subscribes:
  set-db-connection-info:
    Type: "NET_TCP:db:connection-info"
    Required: false
  set-nosql-db-connection-info:
    Type: "NET_TCP:nosqldb:connection-info"
    Required: false
Reservations:
  - MEM >= 10MB
Scaling:
  Min: 1
  Max: -1
```

*jwh: How do we want to cover the manifest.yml features?*

## Cartridge Locking ##

Cartridges on disk within a gear may be either `locked` or `unlocked` at
a given time. When a cartridge is unlocked, a subset of files and directories
(specified by `root_files.txt`) will be writable by the gear user to enable
the cartridge scripts to perform their work.

Unlocking allows the cartridge scripts to have additional access to the gear's
files and directories. Other scripts and hooks written by the end user will
not be able to override decisions you make as the cartridge author.

The lock state is controlled by OpenShift. Cartridges are locked and
unlocked at various points in the cartridge lifecycle.

If you fail to provide a `metadata/root_files.txt` file or the file
is empty, your cartridge will remain always locked. For a very simple cartridges
this may be sufficient.

**Note on security:** Cartridge file locking is not intended to be a
security measure. It is a mechanism to help prevent cartridge users from
inadvertently breaking their application by modifying files reserved
for use by the cartridge.

### Lock configuration ###

The `metadata/root_files.txt` lists the files and directories, one per
line, that will be provided to the cartridge author with read/write
access while the cartridge is unlocked, but only read access to the end
user while the cartridge is locked. Any non-existent files that are
included in the list will be created when the cartridge is unlocked.
Any missing parent directories will be created as needed. The list
is anchored at the gear's home directory.  Entries ending in slash
is processed as a directory.  Entries ending in asterisk are a list
of files.  Entries ending in an other character are considered files.
OpenShift will not attempt to change files to directories or vice versa,
and your cartridge may fail to operate if files are miscatergorized and
you depend on OpenShift to create them.

#### Lock configuration example

Here is a `root_files.txt` for a PHP cartridge:

    .pearrc
    php-5.3/bin/
    php-5.3/bin/*
    php-5.3/conf/*

Note in the above list the files in the `php-5.3/conf` directory are
unlocked but the directory itself is not.  Directories like `.node-gyp`
and `.npm` in nodejs are **NOT** candidates to be created in this manner
as they require the gear to have read and write access. These directories
would need to be created by the nodejs `setup` script which is run while
the gear is unlocked.

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
choose to write these scripts as shim code to setup the necessary
environment before calling additional scripts you write. Or, you may
choose to create symlinks from these names to a name of your choosing.
Your API is the scripts and their associated actions.

A cartridge must implement the following scripts:

* `setup`: prepare this instance of cartridge to be operational
* `teardown`: prepare this instance of cartridge to be removed
* `runhook`: run user provided code
* `control`: command cartridge to report or change state


## bin/setup

##### Synopsis

`setup [--version=<version>]`

##### Options

* `--version=<version>`: Selects which version of cartridge to install. If no version is provided,
the default from `manifest.yml` will be installed.
* `--homedir` provides the parent directory where your cartridge is installed.
If no homedir is provided, the default is OPENSHIFT_HOMEDIR.

##### Description

The `setup` script is responsible for creating and/or configuring the files that
were copied from the cartridge repository into the gear's directory.

Lock context: `unlocked`

##### Messaging to OpenShift from cartridge

Your cartridge may provide a service or services that is consumed by
multiple gears in one application. OpenShift provides the orchestration
necessary for you to publish this service or services. Each message is
written to stdout, one message per line.

* `ENV_VAR_ADD: <variable name>=<value>`
* `CART_DATA: <variable name>=<value>`
* `CART_PROPERTIES: <key>=<value>`
* `APP_INFO: <value>`

*jwh: need to explain when/where to use each...  May need to change format to match Lock context*
*jwh: See Krishna when he is back in town. He wished to make changes in this area with the model refactor.*

## bin/teardown

##### Synopsis

`teardown`

##### Description

The `teardown` script prepares the gear for the cartridge to be
removed. This is not called when the gear is destroyed.  The `teardown`
script is only run when a cartridge is to be removed from the gear.
The gear will continue to operate minus the functionality of this
cartridge.

Lock context: `unlocked`

*jwh: If all future gears are scaled, should teardown just always be called?*

##### Messaging to OpenShift from cartridge

After `teardown` your cartridge's services are no longer available. For
each environment variable you published, you must now un-publish it by
writing to stdout the following message(s):

* `ENV_VAR_REMOVE: <name>`

*jwh: See Krishna when he is back in town. He wished to make changes in this area with the model refactor.*

## bin/runhook

##### Synopsis

`runhook <action> <uuid>`

##### Options

* `action`: which user operation the cartridge should perform
* `uuid`: OPENSHIFT_GEAR_UUID *jwh: Do we still need this? It should always in the environment...*

##### Description

Hooks are called by OpenShift to allow the end user to control aspects
of the cartridge or the software controlled by the cartridge.

The `runhook` script is usually a shim to ensure the environment
is correct for running the `action` code.  Hooks are called by OpenShift
when it requires an action that is or may be dependent on unique
properties of your cartridge or the code your cartridge controls. This
script may be written as a shim to provide access to a SCL-supported
language.

The hooks/scripts provided by the user are installed in
`$OPENSHIFT_HOME/app-root/runtime/repo/.openshift/action-hooks`.
Non-existent or Non-executable scripts must be ignored.

Return an exit status of 0 for successful or unsupported
operations. Non-zero when an error occurres. Text written on stdout may
or may not be reported to the user.  Text returned on stderr will be
logged, again it may or may not be reported to the user.

Actions:

 *jwh: not sure of list yet*

Lock context: `locked`


## bin/control

##### Synopsis

`control <action>`

##### Options

* `action`: which operation the cartridge should perform.

##### Description

The `control` script allows OpenShift or user to control the state of the cartridge.

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
      determine what should be released. Be frugal, on some systems resources may be
      very limited. Some possible actions:
```
    git gc...
    rm .../logs/log.[0-9]
    mvn clean
```

Lock context: `locked`

## bin/build

##### Synopsis

`build`

##### Description

The `build` script is called during the `git push` to perform builds of the user's new code.

Lock context: `locked`


Environment Variables
---------------------

Environment variables are use to communicate setup information between
this cartridge and others, and to OpenShift.  The cartridge controlled
variables are stored in the env directory and will be loaded after
system provided environment variables but before your code is called.
OpenShift provided environment variables will be loaded and available
to be used for all cartridge entry points.

*jwh: ruby 1.9 makes providing environments very easy. We should exploit that.*

### System Provided Variables (Read Only) ###
 * `HISTFILE` bash history file
 * `OPENSHIFT_APP_DNS` the application's fully qualified domain name that your cartridge is a part of
 * `OPENSHIFT_APP_NAME` the validated user assigned name for the application. Black list is system dependent.
 * `OPENSHIFT_APP_UUID` OpenShift assigned UUID for the application
 * `OPENSHIFT_DATA_DIR` the directory where your cartridge may store data
 * `OPENSHIFT_GEAR_DNS` the gear's fully qualified domain name that your cartridge is a part of. May or may not be equal to
                        `OPENSHIFT_APP_DNS`
 * `OPENSHIFT_GEAR_NAME` OpenShift assigned name for the gear. May or may not be equal to `OPENSHIFT_APP_NAME`
 * `OPENSHIFT_GEAR_UUID` OpenShift assigned UUID for the gear
 * `OPENSHIFT_HOMEDIR` OpenShift assigned directory for the gear
 * `OPENSHIFT_INTERNAL_IP` the private IP address for this gear *jwh: may go away*
 * `OPENSHIFT_INTERNAL_PORT` the private PORT for this gear *jwh: may go away*
 * `OPENSHIFT_REPO_DIR` the directory where the software your cartridge controls is stored
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
`PATH=$OPENSHIFT_HOMEDIR/{cartridge&nbsp;name}-{cartridge&nbsp;version}/bin:/bin:/usr/bin/`
when your code is executed. Any additional directories your code requires
will need to be set in your shim code.
