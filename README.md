# Elastic stack (ELK) on Docker

Elastic stack (ELK) on Docker (APM, Logging, Elasticsearch, Kibana, Beats)

Run the latest version of the [Elastic stack][elk-stack] with Docker and Docker Compose.

## Contents

1. [Requirements](#requirements)
   - [Host setup](#host-setup)
   - [SELinux](#selinux)
   - [Docker for Desktop](#docker-for-desktop)
     - [Windows](#windows)
     - [macOS](#macos)
2. [Usage](#usage)
   - [Bringing up the stack](#bringing-up-the-stack)
   - [Cleanup](#cleanup)
   - [Initial setup](#initial-setup)
     - [Setting up user authentication](#setting-up-user-authentication)
     - [Injecting data](#injecting-data)
     - [Default Kibana index pattern creation](#default-kibana-index-pattern-creation)
3. [Configuration](#configuration)
   - [How to configure Elasticsearch](#how-to-configure-elasticsearch)
   - [How to configure Kibana](#how-to-configure-kibana)
   - [How to configure Logstash](#how-to-configure-logstash)
   - [How to disable paid features](#how-to-disable-paid-features)
   - [How to scale out the Elasticsearch cluster](#how-to-scale-out-the-elasticsearch-cluster)
4. [Extensibility](#extensibility)
   - [How to add plugins](#how-to-add-plugins)
   - [How to enable the provided extensions](#how-to-enable-the-provided-extensions)
5. [JVM tuning](#jvm-tuning)
   - [How to specify the amount of memory used by a service](#how-to-specify-the-amount-of-memory-used-by-a-service)
   - [How to enable a remote JMX connection to a service](#how-to-enable-a-remote-jmx-connection-to-a-service)
6. [Going further](#going-further)
   - [Using a newer stack version](#using-a-newer-stack-version)
   - [Plugins and integrations](#plugins-and-integrations)
   - [Swarm mode](#swarm-mode)

## Requirements

### Host setup

- [Docker Engine](https://docs.docker.com/install/) version **17.05+**
- [Docker Compose](https://docs.docker.com/compose/install/) version **1.12.0+**
- 1.5 GB of RAM

By default, the stack exposes the following ports:

- 5000: Logstash TCP input
- 9200: Elasticsearch HTTP
- 9300: Elasticsearch TCP transport
- 5601: Kibana

> :information_source: Elasticsearch's [bootstrap checks][booststap-checks] were purposely disabled to facilitate the
> setup of the Elastic stack in development environments. For production setups, we recommend users to set up their host
> according to the instructions from the Elasticsearch documentation: [Important System Configuration][es-sys-config].

### SELinux

On distributions which have SELinux enabled out-of-the-box you will need to either re-context the files or set SELinux
into Permissive mode in order for docker-elk to start properly. For example on Redhat and CentOS, the following will
apply the proper context:

```console
$ chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

### Docker for Desktop

#### Windows

Ensure the [Shared Drives][win-shareddrives] feature is enabled for the `C:` drive.

#### macOS

The default Docker for Mac configuration allows mounting files from `/Users/`, `/Volumes/`, `/private/`, and `/tmp`
exclusively. Make sure the repository is cloned in one of those locations or follow the instructions from the
[documentation][mac-mounts] to add more locations.

## Usage

### Bringing up the stack

Clone this repository onto the Docker host that will run the stack, then start services locally using Docker Compose:

```console
$ docker-compose up
```

You can also run all services in the background (detached mode) by adding the `-d` flag to the above command.

> :information_source: You must run `docker-compose build` first whenever you switch branch or update a base image.

If you are starting the stack for the very first time, please read the section below attentively.

### Cleanup

Elasticsearch data is persisted inside a volume by default.

In order to entirely shutdown the stack and remove all persisted data, use the following Docker Compose command:

```console
$ docker-compose down -v
```

## Initial setup

### Setting up user authentication

> :information_source: Refer to [How to disable paid features](#how-to-disable-paid-features) to disable authentication.

The stack is pre-configured with the following **privileged** bootstrap user:

- user: _elastic_
- password: _changeme_

Although all stack components work out-of-the-box with this user, we strongly recommend using the unprivileged [built-in
users][builtin-users] instead for increased security.

1. Initialize passwords for built-in users

```console
$ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
```

Passwords for all 6 built-in users will be randomly generated. Take note of them.

2. Unset the bootstrap password (_optional_)

Remove the `ELASTIC_PASSWORD` environment variable from the `elasticsearch` service inside the Compose file
(`docker-compose.yml`). It is only used to initialize the keystore during the initial startup of Elasticsearch.

3. Replace usernames and passwords in configuration files

Use the `kibana` user inside the Kibana configuration file (`kibana/config/kibana.yml`) and the `logstash_system` user
inside the Logstash configuration file (`logstash/config/logstash.yml`) in place of the existing `elastic` user.

Replace the password for the `elastic` user inside the Logstash pipeline file (`logstash/pipeline/logstash.conf`).

> :information*source: Do not use the `logstash_system` user inside the Logstash \_pipeline* file, it does not have
> sufficient permissions to create indices. Follow the instructions at [Configuring Security in Logstash][ls-security]
> to create a user with suitable roles.

See also the [Configuration](#configuration) section below.

4. Restart Kibana and Logstash to apply changes

```console
$ docker-compose restart kibana logstash
```

> :information_source: Learn more about the security of the Elastic stack at [Tutorial: Getting started with
> security][sec-tutorial].

### Injecting data

Give Kibana about a minute to initialize, then access the Kibana web UI by hitting
[http://localhost:5601](http://localhost:5601) with a web browser and use the following default credentials to log in:

- user: _elastic_
- password: _\<your generated elastic password>_

Now that the stack is running, you can go ahead and inject some log entries. The shipped Logstash configuration allows
you to send content via TCP:

```console
# Using BSD netcat (Debian, Ubuntu, MacOS system, ...)
$ cat /path/to/logfile.log | nc -q0 localhost 5000
```

```console
# Using GNU netcat (CentOS, Fedora, MacOS Homebrew, ...)
$ cat /path/to/logfile.log | nc -c localhost 5000
```

You can also load the sample data provided by your Kibana installation.

### Default Kibana index pattern creation

When Kibana launches for the first time, it is not configured with any index pattern.

#### Via the Kibana web UI

> :information*source: You need to inject data into Logstash before being able to configure a Logstash index pattern via
> the Kibana web UI. Then all you have to do is hit the \_Create* button.

Refer to [Connect Kibana with Elasticsearch][connect-kibana] for detailed instructions about the index pattern
configuration.

#### On the command line

Create an index pattern via the Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.5.1' \
    -u elastic:<your generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

The created pattern will automatically be marked as the default index pattern as soon as the Kibana UI is opened for the first time.

## Configuration

> :information_source: Configuration is not dynamically reloaded, you will need to restart individual components after
> any configuration change.

### How to configure Elasticsearch

The Elasticsearch configuration is stored in [`elasticsearch/config/elasticsearch.yml`][config-es].

You can also specify the options you want to override by setting environment variables inside the Compose file:

```yml
elasticsearch:
  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Please refer to the following documentation page for more details about how to configure Elasticsearch inside Docker
containers: [Install Elasticsearch with Docker][es-docker].

### How to configure Kibana

The Kibana default configuration is stored in [`kibana/config/kibana.yml`][config-kbn].

It is also possible to map the entire `config` directory instead of a single file.

Please refer to the following documentation page for more details about how to configure Kibana inside Docker
containers: [Running Kibana on Docker][kbn-docker].

### How to configure Logstash

The Logstash configuration is stored in [`logstash/config/logstash.yml`][config-ls].

It is also possible to map the entire `config` directory instead of a single file, however you must be aware that
Logstash will be expecting a [`log4j2.properties`][log4j-props] file for its own logging.

Please refer to the following documentation page for more details about how to configure Logstash inside Docker
containers: [Configuring Logstash for Docker][ls-docker].

### How to disable paid features

Switch the value of Elasticsearch's `xpack.license.self_generated.type` option from `trial` to `basic` (see [License
settings][trial-license]).

## Extensibility

### How to add plugins

To add plugins to any ELK component you have to:

1. Add a `RUN` statement to the corresponding `Dockerfile` (eg. `RUN logstash-plugin install logstash-filter-json`)
2. Add the associated plugin code configuration to the service configuration (eg. Logstash input/output)
3. Rebuild the images using the `docker-compose build` command

### How to enable the provided extensions

A few extensions are available inside the [`extensions`](extensions) directory. These extensions provide features which
are not part of the standard Elastic stack, but can be used to enrich it with extra integrations.

The documentation for these extensions is provided inside each individual subdirectory, on a per-extension basis. Some
of them require manual changes to the default ELK configuration.

## JVM tuning

### How to specify the amount of memory used by a service

By default, both Elasticsearch and Logstash start with [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) allocated to
the JVM Heap Size.

The startup scripts for Elasticsearch and Logstash can append extra JVM options from the value of an environment
variable, allowing the user to adjust the amount of memory that can be used by each component:

| Service       | Environment variable |
| ------------- | -------------------- |
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

To accomodate environments where memory is scarce (Docker for Mac has only 2 GB available by default), the Heap Size
allocation is capped by default to 256MB per service in the `docker-compose.yml` file. If you want to override the
default JVM configuration, edit the matching environment variable(s) in the `docker-compose.yml` file.

For example, to increase the maximum JVM Heap Size for Logstash:

```yml
logstash:
  environment:
    LS_JAVA_OPTS: -Xmx1g -Xms1g
```

### How to enable a remote JMX connection to a service

As for the Java Heap memory (see above), you can specify JVM options to enable JMX and map the JMX port on the Docker
host.

Update the `{ES,LS}_JAVA_OPTS` environment variable with the following content (I've mapped the JMX service on the port
18080, you can change that). Do not forget to update the `-Djava.rmi.server.hostname` option with the IP address of your
Docker host (replace **DOCKER_HOST_IP**):

```yml
logstash:
  environment:
    LS_JAVA_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## Going further

### Using a newer stack version

To use a different Elastic Stack version than the one currently available in the repository, simply change the version
number inside the `.env` file, and rebuild the stack with:

```console
$ docker-compose build
$ docker-compose up
```

> :information_source: Always pay attention to the [upgrade instructions][upgrade] for each individual component before
> performing a stack upgrade.
