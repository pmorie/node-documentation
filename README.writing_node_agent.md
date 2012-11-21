# How To Write An OpenShift Origin Node Agent 2.0 #

OpenShift node agents provide the interface for the broker to command a gear or gear's cartridge. 

## Gear Directory Structure ##

This is the minimal structure to which your gear is expected to conform
when written to disk. Failure to meet these expections will cause your
gear to not function when either installed or used on OpenShift. You may
not have additional directories or files at the `uuid` level.  They will
interfere with OpenShift's operation.

    .../`uuid`
    +- `cartridge name`-`cartridge version`
    |   +- bin
    |   |  +- setup
    |   |  +- teardown
    |   |  +- runhook
    |   |  +- control
    |   |  +- build
    |   +- versions
    |   |  +- `software version`
    |   |  |  +- bin
    |   |  |     +- build
    |   +- env
    |   +- metadata
    |   |  +- manifest.yml
    |   |  +- root_files.txt
    +- `cartridge name`-`cartridge version`
    |   +- see above...
    +- app-root
    |  +- data
    |  +- runtime
    |     +- repo
    +- git
    +- .ssh
    +- .tmp
    +- .sandbox
    +- .env *jwh: This is where we could put gear-wide environment variables.*

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

### Configuring Locking ###

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

Here is a list for the php cartridge.

    .pearrc
    php-5.3/bin/
    php-5.3/bin/*
    php-5.3/conf/*

Note in the above list the files in the `php-5.3/conf` directory are
unlocked but the directory itself is not.  Directories like `.node-gyp`
and `.npm` for the nodejs are **NOT** candidates to be created in this
manner as they require the gear to have read and write access. These
directories would need to be created by the nodejs setup script which
is run while the gear is unlocked.

The following list of directories are reserved:

    .ssh
    .sandbox
    .tmp
    app-root
    git

## Cartridge Operations ##

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


## High Level Orchestrations

Broker level orchestrations orchestrations which are accomplished by the node library.


### Create Application

* Gear Create
* Cartridge Install
* Expose ports
* Execute connectors


## Node Operation Summary

An overview of the higher level capabilities of the node, with information about how lower-level operations are orchestrated / invoked.


### Gear

* Create
* Delete


### Cartridge

* Configure
* Deconfigure
* Expose ports
* Conceal ports
* Add / remove environment vars
* Start / Stop / Restart (Control cartridge?)

## Node Operation Details

### Gear Create

* Creates a UNIX account representing the gear user.
* Creates skeletal file system
* Creates cgroups / mcslabels?
* Creates port proxy config
* Create standard gear environment variables
* Install default gear httpd conf
* Bounce node httpd

### Gear Delete

* Call cartridge runhook pre-destroy
* Corral and kill all gear user processes
* Call cartridge teardown
* Delete gear directories
* Delete gear user

TODO: deal with node httpd bouncing somewhere

### Cartridge Configure

* Disable cgroups
* Create the initial cart directory from the cart repo/template
* Unlock cartridge
* Call cartridge setup
* Lock cartridge
* Call cartridge control start
* Install cart-supplied node http.d confs
* Bounce the node httpd
* Enable cgroups

TODO: any runhook calls here? e.g. post-install

### Cartridge Deconfigure
* Conceal ports
* Disable cgroups
* Call cartridge control stop
* Unlock cartridge
* Call cartridge teardown
* Uninstall cart-supplied node httpd.confs
* Lock cartridge
* Delete cartridge directory
* Enable cgroups
* Bounce the node httpd (performance impact?)

TODO: any runhook calls here? e.g. post-remove


### Cartridge expose port

* Create .env/OPENSHIFT_${CART_NS}_PROXY_PORT
* Report OPENSHIFT_GEAR_DNS, OPENSHIFT_${CART_NS}_PROXY_PORT, OPENSHIFT_INTERNAL_IP, OPENSHIFT_INTERNAL_PORT back to broker

### Cartridge conceal ports

* Delete .env/OPENSHIFT_${CART_NS}_PROXY_PORT
* remove_proxy_port $uuid "$OPENSHIFT_INTERNAL_IP:$OPENSHIFT_INTERNAL_PORT"

## Low Level

### Lock cartridge

* chown files in root_files.txt
* chcon to set sel context

### Unlock cartridge

* chown files in root_files.txt
* chcon to set sel context
