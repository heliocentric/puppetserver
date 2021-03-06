---
layout: default
title: "Puppet Server: Release Notes"
canonical: "/puppetserver/latest/release_notes.html"
---

[Trapperkeeper]: https://github.com/puppetlabs/trapperkeeper
[service bootstrapping]: ./configuration.markdown#service-bootstrapping
[auth.conf]: ./config_file_auth.markdown
[puppetserver.conf]: ./config_file_puppetserver.markdown
[product.conf]: ./config_file_product.markdown

For release notes on versions of Puppet Server prior to Puppet Server 2.5, see [docs.puppet.com](https://docs.puppet.com/puppetserver/2.4/release_notes.html).

## Puppet Server 2.7.1

Released November 21, 2016.

This is a bug-fix release of Puppet Server.

> **Warning:** If you're upgrading from Puppet Server 2.4 or earlier and have modified `bootstrap.cfg`, `/etc/sysconfig/puppetserver`, or `/etc/default/puppetserver`, see the [Puppet Server 2.5 release notes first](#potential-breaking-issues-when-upgrading-with-a-modified-bootstrapcfg) **before upgrading** for instructions on avoiding potential failures.

### Bug Fix: Set `puppetserver gem` Java arguments separately from the Server service

In Puppet Server 2.7.0, the `JAVA_ARGS` from Puppet Server's sysconfig/default file (typically located at `/etc/sysconfig/puppetserver` or `/etc/defaults/puppetserver`) were passed along to the Java process started when running the [`puppetserver gem`](./gems.markdown) command. This could lead to arguments that are intended only for use when running the full puppetserver service --- for example, debug arguments or large memory heap settings --- being used when running `gem` commands, which could cause the `gem` commands to fail.

In Puppet Server 2.7.1, you can set custom arguments to be passed into the Java process for the `gem` command via the new `JAVA_ARGS_CLI` environment variable, either temporarily on the command line or persistently by adding it to the sysconfig/default file. The `JAVA_ARGS_CLI` environment variable also controls the arguments used when running the `puppetserver ruby` and `puppetserver irb` [subcommands](./subcommands.markdown).

-   [SERVER-1644](https://tickets.puppetlabs.com/browse/SERVER-1644)

## Puppet Server 2.7.0

Released November 8, 2016.

This is a feature and bug-fix release of Puppet Server.

> **Warning:** If you're upgrading from Puppet Server 2.4 or earlier and have modified `bootstrap.cfg`, `/etc/sysconfig/puppetserver`, or `/etc/default/puppetserver`, see the [Puppet Server 2.5 release notes first](#potential-breaking-issues-when-upgrading-with-a-modified-bootstrapcfg) **before upgrading** for instructions on avoiding potential failures.

### New Feature: Disable update checking and telemetry data collection

Puppet Server automatically communicates with Puppet's servers to check for updates. Puppet Server 2.7 adds the option to stop checking for updates by creating a new configuration file, [`product.conf`][product.conf], setting `check-for-updates` to false, then [restarting Puppet Server](./restarting.markdown).

For more information, see the [`product.conf`][product.conf] documentation.

-   [SERVER-1599](https://tickets.puppetlabs.com/browse/SERVER-1599)

### New Feature: New `reload` service action for faster, safer service restarts

Since Puppet Server 2.3, administrators could send a HUP signal to the `puppetserver` process to [quickly reload the service](./restarting.markdown). This provides a faster way to apply changes to settings that require a Puppet Server restart, but sending the signal directly to the process could lead to potential conflicts with actions attempted while the service was reloading.

Puppet Server 2.7 adds a `reload` action can be performed via the operating system's service framework (for example, running `service puppetserver reload`) to perform a HUP reload without requiring a Java process restart. The result is similar to sending a SIGHUP directly to the process, but with the additional benefits of waiting until the server has been reloaded before performing additional scripted commands, tracking the process's ID for you, and providing a more informative exit code should the service fail to reload.

For details, see [Restarting Puppet Server](./restarting.markdown).

-   [SERVER-1490](https://tickets.puppetlabs.com/browse/SERVER-1490)

### Bug Fix: Fix attribute order in CA certificates

In Puppet Server 2.6 and earlier, when the server's certificate authority (CA) service issued a client certificate from a CA with multiple attributes in its certificate Subject's distinguished name (DN), the attributes in the client certificate's Issuer DN were in reverse order from the corresponding attributes in the CA certificate's Subject DN.

For example, if the Subject DN in the CA certificate were `/C=US/CN=myca.org`, the Issuer DN in the client certificate would be `/CN=myca.org/C=US`. The improper attribute order causes SSL connections made with the client certificate to fail validation.

In Puppet Server 2.7, the Issuer DN attributes for newly generated client certificates are formatted in the same order as in the corresponding CA Subject DN. SSL connections made with these new certificates are now validated, allowing for successful secure connections.

-   [SERVER-1545](https://tickets.puppetlabs.com/browse/SERVER-1545)

### Experimental Feature: Run Puppet Server in an MRI 2.0 compatibility mode

Puppet Server uses JRuby 1.7 configured in a "1.9" MRI compatibility mode. In Puppet Server 2.6 and earlier, this configuration could not be changed.

In Puppet Server 2.7, you can choose to run JRuby in modes compatible with 1.9 or 2.0 by changing the `compat-version` setting in [`puppetserver.conf`][puppetserver.conf]. If set to "2.0", users can install and use gems that require Ruby 2.0 with Puppet Server.

> **Warning:** This is an experimental feature and might not be suitable for production.

-   [SERVER-1585](https://tickets.puppetlabs.com/browse/SERVER-1585)

### Bug Fix: Honor all `pp_` custom certificate extension short names

In Puppet Server 2.6, the Puppet Server certificate authority (CA) did not honor short names for these `pp_*` custom certificate extensions in Puppet:

-   pp_region
-   pp_datacenter
-   pp_zone
-   pp_network
-   pp_securitypolicy
-   pp_cloudplatform
-   pp_apptier
-   pp_hostname

Puppet Server 2.7 honors these short names as expected.

-   [SERVER-1583](https://tickets.puppetlabs.com/browse/SERVER-1583)

### Bug Fix: Make `generate()` function behavior consistent with MRI/Rack Puppet masters

When a command executed by Puppet's [`generate()` function](https://docs.puppet.com/puppet/latest/reference/function.html#generate) returns a non-zero exit code, the MRI/Rack Puppet master throws an exception while retrieving the catalog similar to:

```
Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Failed to execute generator /bin/false: Execution of '/bin/false' returned 1:  at /etc/puppet/environments/production/manifests/test.pp:2 on node MYNODE.mygtld
```

However, Puppet Server 2.6 and earlier do not throw an exception as expected, and instead appears to apply the catalog without error, making Puppet Server's behavior inconsistent with the MRI/Rack master.

Puppet Server 2.7 resolves this issue by throwing the expected exception.

Also, in Puppet Server 2.6 and earlier, `generate()` doesn't merge the executed command's output from `STDERR` to `STDOUT` as expected. Puppet Server 2.7 resolves this issue by including the `STDERR` output.

-   [SERVER-1570](https://tickets.puppetlabs.com/browse/SERVER-1570): puppet4 function generate() should throw exception when command fails
-   [SERVER-1571](https://tickets.puppetlabs.com/browse/SERVER-1571): The function generate() should merge stdout and stderr

### Bug Fix: Avoid "partial state" error if an agent attempts a Puppet run on the master before first puppetserver service start

In Puppet Server 2.6 and earlier, if the agent on a Puppet Server master starts a Puppet run (such as `puppet agent -t`) before the Puppet Server service has first started, private and public keys would be created for the agent but the Puppet Server service would subsequently fail to start with an error message similar to:

```
java.lang.IllegalStateException: Cannot initialize master with partial state; need all files or none.
Found:
/var/lib/puppet/ssl/private_keys/master.pem
Missing:
/var/lib/puppet/ssl/certs/master.pem
```

In this situation, Puppet Server 2.7 now uses pre-generated public and private keys to generate a certificate for the master and will finish starting without error.

-   [SERVER-528](https://tickets.puppetlabs.com/browse/SERVER-528)

### New Feature: Required gems are packaged with Puppet Server, with a new `GEM_PATH` and setting

Some gems are required by both the Puppet agent and Puppet Server. To ship these gems as part of our packaging, Puppet Server 2.7 changes how Puppet Server looks for gems.

In Puppet Server 2.6 and earlier, Puppet Server's `GEM_PATH` was comprised of only a single directory by default: `/opt/puppetlabs/server/data/puppetserver/jruby-gems`. Puppet Server also used this directory as the value for `GEM_HOME`, meaning that the `puppetserver gem install` command installed gems to this directory.

In Puppet Server 2.7, we've added a second path to `GEM_PATH`: `/opt/puppetlabs/server/data/puppetserver/vendored-jruby-gems`. Gems that are known to be needed by Puppet Server will be installed in this directory as part of the Puppet Server packaging.

`GEM_HOME` still points to the same `jruby-gems` directory as it did in previous releases, and `puppetserver gem install` continues to install gems to, and use gems from, that directory.

To configure the `GEM_PATH`, set the new `gem-path` setting in [`puppetserver.conf`][puppetserver.conf].

-   [SERVER-1412](https://tickets.puppetlabs.com/browse/SERVER-1412)

### Bug Fix: `puppetserver gem` command makes installed gems readable

In Puppet Server 2.6 and earlier, if the system's default umask did not permit world-readability for gems installed with the 'puppetserver gem' subcommand, the `puppetserver` process might not be able to use the resulting gemspec files, leading to errors such as:

```
Exception in thread "main" org.jruby.exceptions.RaiseException: (LoadError) no such file to load -- trollop
Exception in thread "main" org.jruby.exceptions.RaiseException: (Errno::EACCES) /opt/puppetlabs/server/data/puppetserver/jruby-gems/specifications/trollop-2.1.2.gemspec
```

Puppet Server 2.7 resolves this issue by explicitly setting a umask of 0022 when running any `puppetserver gem` subcommand. This ensures that `puppetserver` can use any gems installed by the `gem` subcommand at run-time.

-   [SERVER-1601](https://tickets.puppetlabs.com/browse/SERVER-1601)

### Other new features

-   [SERVER-1589](https://tickets.puppetlabs.com/browse/SERVER-1589): Use `.gz` extensions for Puppet Server log file archives, and rotate them when they reach 200MB in size instead of 10MB.

## Puppet Server 2.6

Released September 8, 2016.

This is a feature and bug-fix release of Puppet Server. This release also adds an official Puppet Server package for SuSE Enterprise Linux (SLES) 12.

> **Warning:** If you're upgrading from Puppet Server 2.4 or earlier and have modified `bootstrap.cfg`, `/etc/sysconfig/puppetserver`, or `/etc/default/puppetserver`, see the [Puppet Server 2.5 release notes first](#potential-breaking-issues-when-upgrading-with-a-modified-bootstrapcfg) **before upgrading** for instructions on avoiding potential failures.

### New feature: JVM metrics endpoint `/status/v1/services`

Puppet Server provides a new endpoint, `/status/v1/services`, which can provide basic Java Virtual Machine-level metrics related to the current Puppet Server process's memory usage.

To request this data, make an HTTP GET request to Puppet Server with a query string of `level=debug`. For details on the endpoint and its response, see the [Services endpoint documentation](./status-api/v1/services.markdown).

> **Experimental feature note:** These metrics are experimental. The names and values of the metrics may change in future releases.

-   [SERVER-1502](https://tickets.puppetlabs.com/browse/SERVER-1502)

### New feature: Logback replaces logrotate for Server log rotation

Previous versions of Puppet Server would rotate and compress logs daily using logrotate. Puppet Server 2.6 uses Logback, the logging library used by Puppet Server's Java Virtual Machine (JVM).

Under logrotate, certain pathological error states --- such as running out of file handles --- could cause previous versions of Puppet Server to fill up disk partitions with logs of stack traces.

In Puppet Server 2.6, Logback compresses Server-related logs into archives when their size exceeds 10MB. Also, when the total size of all Puppet Server logs exceeds 1GB, Logback deletes the oldest logs. These improvements should limit the space that Puppet Server's logs consume and prevent them from filling partitions.

> **Debian upgrade note:** On Debian-based Linux distributions, logrotate will continue to attempt to manage your Puppet Server log files until `/etc/logrotate.d/puppetserver` is removed. These logrotate attempts are harmless, but will generate a duplicate archive of logs. As a best practice, delete `puppetserver` from `logrotate.d` after upgrading to Puppet Server 2.6.
>
> This doesn't affect clean installations of Puppet Server on Debian, or any upgrade or clean installation on other Linux distributions.

-   [SERVER-366](https://tickets.puppetlabs.com/browse/SERVER-366)

### Bug fixes: Update JRuby to resolve several issues

This release resolves two issues by updating the version of JRuby used by Puppet Server to 1.7.26.

In previous versions of Puppet Server 2.x, when a variable lookup is performed from Ruby code or an ERB template and the variable is not defined, catalog compilation could periodically fail with an error message similar to:

```
Puppet Evaluation Error: Error while evaluating a Resource Statement, Evaluation Error: Error while evaluating a Function Call, integer 2181729414 too big to convert to `int` at <PUPPET FILE>
```

The error message is inaccurate; the lookup should return nil. The error is a [bug in JRuby](https://github.com/jruby/jruby/issues/3980), which Puppet Server uses to run Ruby code. Puppet Server 2.6 resolves this by updating JRuby.

-   [SERVER-1408](https://tickets.puppetlabs.com/browse/SERVER-1408)

Also, when Puppet Server uses a large JVM memory heap and large number of JRuby instances, Puppet Server could fail to start and produce error messages in the `puppetserver.log` file similar to:

```
java.lang.IllegalStateException: There was a problem adding a JRubyPuppet instance to the pool.
Caused by: org.jruby.embed.EvalFailedException: (LoadError) load error: jopenssl/load -- java.lang.NoClassDefFoundError: org/jruby/ext/openssl/NetscapeSPKI
```

We [fixed the underlying issue in JRuby](https://github.com/jruby/jruby/pull/4063), and this fix is included in Puppet Server 2.6.

- [SERVER-858](https://tickets.puppetlabs.com/browse/SERVER-1408)

### New feature: Whitelist Ruby environment variables

Puppet Server 2.6 adds the ability to specify a whitelist of environment variables made available to Ruby code. To whitelist variables, add them to the `environment-vars` section under the `jruby-puppet` configuration section in [`puppetserver.conf`][puppetserver.conf].

- [SERVER-584](https://tickets.puppetlabs.com/browse/SERVER-584)

## Puppet Server 2.5

Released August 11, 2016.

This is a feature and bug-fix release of Puppet Server.

> ### Potential breaking issues when upgrading with a modified `bootstrap.cfg`
>
> If you disabled the certificate authority (CA) on Puppet Server by editing the [`bootstrap.cfg`][service bootstrapping] file on older versions of Puppet Server --- for instance, because you have a multi-master configuration with the default CA disabled on some masters, or use an external CA --- be aware that Puppet Server as of version 2.5.0 no longer uses the `bootstrap.cfg` file.
>
> Puppet Server 2.5.0 and newer instead create a new configuration file, `/etc/puppetlabs/puppetserver/services.d/ca.cfg`, if it doesn't already exist, and this new file enables CA services by default.
>
> To ensure that CA services remain disabled after upgrading, create the `/etc/puppetlabs/puppetserver/services.d/ca.cfg` file with contents that disable the CA services _before_ you upgrade to Server 2.5.0 or newer. The `puppetserver` service restarts after the upgrade if the service is running before the upgrade, and the service restart also reloads the new `ca.cfg` file.
>
> Also, back up your masters' [`ssldir`](https://docs.puppet.com/puppet/latest/reference/dirs_ssldir.html) (or at least your `crl.pem` file) _before_ you upgrade to ensure that you can restore your previous certificates and certificate revocation list, so you can restore them in case any mistakes or failures to disable the CA services in `ca.cfg` lead to a master unexpectedly enabling CA services and overwriting them.
>
> For more details, including a sample `ca.cfg` file that disables CA services, see the [bootstrap upgrade notes](./bootstrap_upgrade_notes.markdown).

> ### Potential service failures when upgrading with a modified init configuration
>
> If you modified the init configuration file --- for instance, to [configure Puppet Server's JVM memory allocation](./install_from_packages.html#memory-allocation) or [maximum heap size](./tuning_guide.html) --- and upgrade Puppet Server 2.5.0 or newer with a package manager, you might see a warning during the upgrade that the updated package will overwrite the file (`/etc/sysconfig/puppetserver` in Red Hat and derivatives, or `/etc/default/puppetserver` in Debian-based systems).
>
> The changes to the file support the new service bootstrapping behaviors. If you don't accept changes to the file during the upgrade, the puppetserver service fails and you might see a `Service ':PoolManagerService' not found` or similar warning. To resolve the issue, set the `BOOTSTRAP_CONFIG` setting in the init configuration file to:
>
>     BOOTSTRAP_CONFIG="/etc/puppetlabs/puppetserver/services.d/,/opt/puppetlabs/server/apps/puppetserver/config/services.d/"
>
> If you modified other settings in the file before upgrading, and then overwrite the file during the upgrade, you might need to reapply those modifications after the upgrade.

### New feature: Flexible service bootstrapping/CA configuration file

To disable the Puppet CA service in previous versions of Puppet Server 2.x, users edited the [`bootstrap.cfg`][service bootstrapping] file, usually located at `/etc/puppetlabs/puppetserver/bootstrap.cfg`.

This workflow could cause problems for users performing package upgrades of Puppet Server where `bootstrap.cfg` was modified, because the package might overwrite the modified `bootstrap.cfg` and undo their changes.

To improve the upgrade experience for these users, Puppet Server 2.5.0 can load the service bootstrapping settings from multiple files. This in turn allows us to provide user-modifiable settings in a separate file and avoid overwriting any changes during an upgrade.

-   [SERVER-1470](https://tickets.puppetlabs.com/browse/SERVER-1470)
-   [SERVER-1247](https://tickets.puppetlabs.com/browse/SERVER-1247)
-   [SERVER-1213](https://tickets.puppetlabs.com/browse/SERVER-1213)

### New feature: Signing CSRs with OIDs from a new arc

Puppet Server 2.5.0 can sign certificate signing requests (CSRs) from Puppet 4.6 agents that contain a new custom object identifier (OID) arc to represent secured extensions for use with [`trapperkeeper-authorization`][Trapperkeeper].

> **Aside:** Trapperkeeper powers the [HOCON `auth.conf` and authorization methods][auth.conf] introduced in Puppet Server 2.2.0. This new CSR-signing functionality in Server 2.5.0 builds on features added to Puppet 4.6 and the addition of X.509 extension-based authorization rules added to Trapperkeeper alongside Puppet Server 2.4.

To sign CSRs wth the new OID arc via the Puppet 4.6 command-line tools, use the `puppet cert sign --allow-authorization-extensions` command. See the [`puppet cert` man page](https://docs.puppet.com/puppet/4.6/reference/man/cert.html) for details. This workflow is similar to signing DNS alt names.

The new OID arc is "puppetlabs.1.3", with a long name of "Puppet Authorization Certificate Extension" and short name of `ppAuthCertExt` (where "puppetlabs" is our registered OID arc 1.3.6.1.4.1.34380). Set the extension "puppetlabs.1.3.1" (`pp_authorization`) on CSRs that need to be authenticated via the new workflow. We've also included an default alias of `pp_auth_role` at extension "puppetlabs.1.3.13" for common workflows. See [the Puppet CSR attributes and certificate extensions documentation](https://docs.puppet.com/puppet/4.6/reference/ssl_attributes_extensions.html) for more information.

We've also improved the CLI output of `puppet cert list` and `puppet cert sign` to work better with the `--human-readable` and `--machine-readable` flags, and we allow administrators to force a prompt when signing certificates with the `--interactive` flag.

This allows for easier automated failover to authorized nodes within a Puppet infrastructure and provides tools for creating new, securely automated workflows, such as automated component promotions within Puppet-managed infrastructure.

-   [SERVER-1305](https://tickets.puppetlabs.com/browse/SERVER-1305)

### Bug fix: Unrecognized parse-opts

Puppet Server 2.4.x used a deprecated API for a Clojure CLI option-parsing library. As a result, calls to `puppetserver gem` (either directly, or indirectly by using a `puppetserver_gem` package resource) generated unexpected warning messages:

    Warning: Could not match Warning: The following options to parse-opts are unrecognized: :flag

Puppet Server 2.5.0 updates this library, which prevents this error message from appearing.

-   [SERVER-1378](https://tickets.puppetlabs.com/browse/SERVER-1378)

### Bug fix: Puppet Server no longer ships with an empty PID file

When installed on CentOS 6, Puppet Server 2.4.x included an empty PID file. When running `service puppetserver status`, Puppet Server returned an unexpected error message: `puppetserver dead but pid file exists`.

When performing a clean installation of Puppet Server 2.5.0, no PID file is created, and `service puppetserver status` should return the expected `not running` message.

-   [SERVER-1455](https://tickets.puppetlabs.com/browse/SERVER-1455)
-   [EZ-84](https://tickets.puppetlabs.com/browse/EZ-84)

### Other changes

-   [SERVER-1310](https://tickets.puppetlabs.com/browse/SERVER-1310): Error messages in Puppet Server 2.5.0 use the same standard types as other Puppet projects.
-   [SERVER-1121](https://tickets.puppetlabs.com/browse/SERVER-1121): JRuby pool management code for the [Trapperkeeper Webserver Service][Trapperkeeper] is now its own open-source project, [puppetlabs/jrubyutils](https://github.com/puppetlabs/jruby-utils).
