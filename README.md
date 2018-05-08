nomadns
=======

Nomadns is a small tool that makes it possible to run hierarchical jobs within
[Nomad](https://nomadproject.io). The basic unit of work is a collection of
nomad jobs in the form of a directory. That collection can itself contain
sub-collections that can be configured by the parent collection.

Consider each collection like a function that can call sub-collections or other
functions to perform a task.

Nomadns is a pre-processing tool that templates nomad job using environment
variables.

Collections
-----------

A collection is a directory ending with `.nomad`. It can contain:

- nomad jobs: files ending with `.nomad`
- sub-collection: directory ending with `.nomad`
- environment variable definitions: files ending with `.env` with the same
  prefix as either a nomad job or a sub-collection

Variables are defined from the outer collection first to the inner collection
last. The innter variables takes precedence locally over outer variables.

For a specific nomad job file
`outer-collection.nomad/inner-collection.nomad/job.nomad`, variables are
evaluated in that order:

- `outer-collection.nomad/inner-collection.nomad/job.env`
- `outer-collection.nomad/inner-collection.env`
- `outer-collection.env`
- variables defined when `nomadns` is invoked


Templating
----------

Environment files and nomad jobs can contain templating instructions. Those are
simplistic as they are evaluated using shell script. No scripting language was
used to allow running `nomadns` on CoreOS.

Template variables are substituted using the `[[ENV_NAME]]` syntax.

It is also possible to source external files within templates using the
`[[file(filename)]]` syntax where `filename` is the file you want to include
verbatim.

Environment Files
-----------------

Environment variable files contain one variable per line with name and value
separated by the `=` character with no quoting and no space around the `=` sign.

Special Variables
-----------------

When a collection is instanciated, a token is generated for it. This token can
be used to namespace nomad jobs by using the token as a prefix. These tokens can
be accessed using special environment variables. An outer collection can know
the namespace identifier of an inner collection, but not the other way around.

Those variables are:

- `NS`: unique token for the current collection
- `NS_*FOO*`: the namespace token for the `*foo*.nomad` subcollection. The
  variable name is uppercase while the sub-collection should be lowercase in the
  filesystem.

Invocation
----------

Invoke the `collection-name.nomad` collection with default variables defined aas
multiple `VAR=val` arguments using:

    nomadns collection-name.nomad [VAR=val...]

It will geenrate the nomad jobs and execute those jobs using `nomad run`. The
nomad executable can be overriden using the `NOMAD` environment variable.

Examples
========

As an example, we have a mail system organized this way:

- `main.nomad/`: the main collection for the complete system
    - `mailu.env`: common environment variables for `mailu.nomad`
    - `mailu.nomad/`: collection containing the imap and smtp jobs
        - `imap.nomad`: nomad job serving IMAP
        - `smtp.nomad`: nomad job serving SMTP
        - `env.template`: file included from templates
    - `backup.nomad`: backup nomad job that saves the mail to a backup location
    - `mailu-config.nomad`: one time job that is executed and populates consul
      with variables that configures the downstream nomad jobs

Sample `main.nomad/mailu.env` contain:

    VERSION=latest

Sample `main.nomad/mailu.nomad/env.template` contain:

    # Common variables to be included in consul-template environment 
    BIND_ADDRESS = "::1"
    POSTMASTER = "admin"

    SECRET_KEY="{{ keyOrDefault (printf "ns/%s/secret_key" (env "CONSUL_NAMESPACE_ID")) "ChangeMeChangeMe"}}"

Sample `main.nomad/mailu.nomad/imap.nomad` contain:

    job "[[NS]]-imap" {
      datacenters = ["dc-1"]
      group "dovecot" {
        count = 1
        task "dovecot" {
          driver = "docker"
          config {
            image = "mailu/dovecot:[[VERSION]]"
            volumes = [
              "/var/lib/mailu-[[NS]]/mail:/mail",
            ]
          }
          env {
            "CONSUL_NAMESPACE_ID" = "[[NS]]"
          }
          template {
            env = true
            destination = "env"
            data = <<EOF
    [[file(env.template)]]

    {{ range service "[[NS]]-smtp" }}
    SMTP_ADDRESS = "{{ .Address }}"{{ end }}
    EOF
          }
          service {
            port = 143
            name = "[[NS]]-imap"
            address_mode = "driver"
          }
        }
      }
    }

Sample `main.nomad/mailu-config.nomad` contains:

    job "[[NS]]-mailu-config" {
      datacenters = ["dc-1"]
      type = "batch"
      group "config" {
        count = 1
        task "config" {
          driver = "raw_exec"
          config {
            command = "sh"
            args    = ["config.sh"]
          }
          env {
            "CONSUL_NAMESPACE_ID" = "[[NS]]"
          }
          template {
            destination = "config.sh"
            data = <<EOF
    export PATH="$PATH:/opt/bin"
    consul kv put "ns/[[NS_MAILU]]/tls_flavor" "letsencrypt"
    consul kv put "ns/[[NS_MAILU]]/hostnames"  "mailu.example.org"
    consul kv put "ns/[[NS_MAILU]]/secret_key" "[[SECRET_KEY]]"
    consul kv put "ns/[[NS_MAILU]]/domain"     "mailu.example.org"

    consul kv put "ns/[[NS]]/backup_ftp_password"  "[[BACKUP_FTP_PASSWORD]]"
    consul kv put "ns/[[NS]]/backup_data_password" "[[BACKUP_DATA_PASSWORD]]"
    EOF
          }
        }
      }
    }

Sample `main.nomad/backup.nomad` contains:

    job "[[NS]]-mailu-backup" {
      datacenters = ["dc-1"]
      type = "batch"
      periodic {
        cron             = "@daily"
        prohibit_overlap = true
      }
      group "backup" {
        count = 1
        task "backup" {
          driver = "docker"
          config {
            image = "wernight/duplicity"
            command = "sh"
            args = [ "${NOMAD_TASK_DIR}/backup-script /data/mailu-[[NS_MAILU]]/" ]
            volumes = [
              "/var/lib/mailu-[[NS_MAILU]]/:/data/mailu-[[NS_MAILU]]/",
            ]
          }
          ...
          template {
            env = true
            destination = "env"
            data = <<EOF
    LOCATION     = "{{ key (printf "ns/%s/backup_location" (env "CONSUL_NAMESPACE_ID"))}}"
    FTP_PASSWORD = "{{ key (printf "ns/%s/backup_ftp_password" (env "CONSUL_NAMESPACE_ID"))}}"
    PASSPHRASE   = "{{ key (printf "ns/%s/backup_data_password" (env "CONSUL_NAMESPACE_ID"))}}"
    EOF
          }
        }
      }
    }


