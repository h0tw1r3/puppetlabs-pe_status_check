# Puppet Status Check

- [Description](#description)
- [Setup](#setup)
  - [What this module affects](#what-this-module-affects)
  - [Setup requirements](#setup-requirements)
- [Usage](#usage)
  - [Enable infrastructure checks](#enable-infrastructure-checks)
  - [Disable](#disable)
- [Reporting Options](#reporting-options)
  - [Class declaration](#class-declaration)
  - [Using a Puppetdb Query to report status.](#using-a-puppetdb-query-to-report-status)
  - [Ad-hoc Report (Plans)](#ad-hoc-report-plans)
    - [Setup Requirements](#setup-requirements-1)
    - [Running the plans](#running-the-plans)
- [Reference](#reference)
  - [Fact: puppet_status_check_role](#fact-puppet_status_check_role)
  - [Fact: puppet_status_check](#fact-puppet_status_check)
  - [Fact: agent_status_check](#fact-agent_status_check)
- [How to report an issue or contribute to the module](#how-to-report-an-issue-or-contribute-to-the-module)

## Description

Puppet Status Check provides a way to alert the end-user when Puppet is not in an ideal state. It uses pre-set indicators and has a simplified output that directs the end-user to the next steps for resolution.

## Setup

### What this module affects

This module primarily provides status indicators the fact named `puppet_status_check`. Once nodes have been classified with the module, facts will be generated and the optional indicators can occur. By default, fact collection is set to only check the status of the Puppet agent. Puppet infrastructure checks require additional configuration.

### Setup requirements

Install the module, plug-in sync will be used to deliver the required facts for this module, to each agent node in the environment the module is installed in.

## Usage

Classify nodes with `puppet_status_check`. Notify resources will be added to a node on each Puppet run if indicator's are reporting as `false`. These can be viewed in the Puppet report for each node, or queried from Puppetdb.

### Enable infrastructure checks

The default fact population will not perform checks related to puppet infrastructure services such as the puppetserver, puppetdb, or postgresql. To enable the checks for Puppet servers, set the following parameter to those infrastructure node(s):

```
puppet_status_check::role: primary
```

### Disable

To completely disable the collection of `puppet_status_check` facts, uninstall the module or [classify the module](#class-declaration) with the `enabled` parameter:

```
puppet_status_check::enabled: false
```

## Reporting Options

### Class declaration

To enable fact collection and configure notifications, classify nodes with the `puppet_status_check` class. Examples using `site.pp`:

Check basic agent status:

```puppet
node 'node.example.com' {
  include 'puppet_status_check'
}
```

Check puppet server infrastructure status:

```puppet
node 'node.example.com' {
  class { 'puppet_status_check':
    role => 'primary',
  }
}
```

For maximum coverage, report on all default indicators. However, if you need to make exceptions for your environment, classify the array parameter `indicator_exclusions` with a list of all the indicators you do not want to report on.

```puppet
class { 'puppet_status_check':
  indicator_exclusions => ['S0001','S0003','S0003','S0004'],
}
```

### Using a Puppetdb Query to report status.

As the module uses Puppet's existing fact behavior to gather the status data from each of the agents, it is possible to use PQL (puppet query language) to gather this information.

Consult with your local Puppet administrator to construct a query suited to your organizational needs. 
Please find some examples of using the [puppetdb_cli gem](#https://github.com/puppetlabs/puppetdb-cli) to query the status check facts below:

1. To find the complete output from all nodes listed by certname (this could be a large query based on the number of agent nodes, further filtering is advised ):
   ```shell
   puppet query 'facts[certname,value] { name = "puppet_status_check" }'
   ```
        
2. To find the complete output from all nodes listed by certname with the `primary` role:
   ```shell
   puppet query 'facts[certname,value] { name = "puppet_status_check" and certname in facts[certname] { name = "puppet_status_check_role" and value = "primary" } }'
   ```

3. To find those nodes with a specific status check set to false:
   ```shell
   puppet query 'inventory[certname] { facts.puppet_status_check.S0001 = false }'
   ```

### Ad-hoc Report (Plans)

The plan `puppet_status_check::summary` summarizes the status of each of the checks on target nodes that have the `puppet_status_check` fact enabled. Sample output can be seen below:

**TBC**

#### Setup Requirements

`puppet_status_check::summary` utilize [hiera](https://puppet.com/docs/puppet/latest/hiera_intro.html) to lookup test definitions, this requires placing a static hierarchy in your **environment level** [hiera.yaml](https://puppet.com/docs/puppet/latest/hiera_config_yaml_5.html).

```yaml
plan_hierarchy:
  - name: "Static data"
    path: "static.yaml"
    data_hash: yaml_data
```

See the following [documentation](https://puppet.com/docs/bolt/latest/hiera.html#outside-apply-blocks) for further explanation.

#### Using Static Hiera data to populate indicator_exclusions when executing plans

Place the plan_hierarchy listed in the step above, in the environment layer.

Create a [static.yaml] file in the environment layer hiera data directory
```yaml
puppet_status_check::indicator_exclusions:                                             
  - '<TEST ID>'                                                                
``` 

Indicator ID's within array will be excluded when `running puppet_status_check::summary`.

#### Running the plans

The `puppet_status_check::summary` plans can be run from the [Puppet Bolt](https://puppet.com/bolt). More information on the parameters in the plan can be seen in the [REFERENCE.md](REFERENCE.md).

Example call from the command line to run `puppet_status_check::summary` against all infrastructure nodes:

```shell
bolt plan run puppet_status_check::summary role=primary
```
Example call from the command line to run `puppet_status_check::summary` against all regular agent nodes:

```shell
bolt plan run puppet_status_check:summary role=agent
```

Example call from the command line to run against a set of infrastructure nodes:

```shell
bolt plan run puppet_status_check::summary targets=server-70aefa-0.region-a.domain.com,psql-70aefa-0.region-a.domain.com
```

Example call from the command line to exclude indicators for `puppet_status_check::infra_summary`:

```shell
bolt plan run puppet_status_check::summary -p '{"indicator_exclusions": ["S0001","S0021"]}'
```
Example call from the command line to exclude indicators for `puppet_status_check::agent_summary`:

```shell
bolt plan run puppet_status_check::summary -p '{"indicator_exclusions": ["AS001","AS002"]}'
```

## Reference

### Fact: puppet_status_check_role

This fact is used to determine which status checks are included on an infrastructure node. Classify the `puppet_status_check` module with a `role` parameter to change the role.

| Role     | Description |
| -------- | - |
| primary  | The node hosts a puppetserver, puppetdb, database, and certificate authority |
| compiler | The node hosts a puppetserver and puppetdb |
| postgres | The node hosts a database |
| agent    | The node runs a puppet agent service |

_The role is `agent` by default. _

### Fact: puppet_status_check

This fact is confined to run on infrastructure nodes only.

Refer  below for next steps when any indicator reports a `false`.

| Indicator ID | Description                                                                        | Self-service steps                                                                                                                                                                                                                                                                                                                                                                                                                                                          | What to include in a Support ticket                                                                                                                                                                                                   |
|--------------|------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| S0001        | Determines if the puppet service is running on agents.                                      | See [documentation](https://portal.perforce.com/s/article/Why-is-puppet-service-not-running)                                                                                                                                                                                                                                                                                                                                                                                                 | If the service fails to start, open a Support ticket referencing S0001, and provide `syslog` and any errors output when attempting to restart the service.                                           |                                       |
| S0002        | Determines if the pxp-agent service is running.                                         | Start the pxp-agent service - `puppet resource service pxp-agent ensure=running`, if the service has failed check the logs located in `/var/logs/puppetlabs/pxp-agent`, for information to debug and understand what the logs mean, see the following links for assistance.          (Connection Type Issues)[https://portal.perforce.com/s/article/4442390587671], if the service is up and running but issues still occur, see (Debug Logging)[https://portal.perforce.com/s/article/7606830611223] | If the service fails to start, open a Support ticket referencing S0002, provide `syslog` any errors output when attempting to restart the service, and `/var/log/puppetlabs/pxp-agent/pxp-agent.log` |
| S0003        | Determines if infrastructure components are running in noop.                       | Do not routinely configure noop on PE infrastructure nodes, as it prevents the management of key infrastructure settings. [Disable this setting on infrastructure components.](https://puppet.com/docs/puppet/latest/configuration.html#noop)                                                                                                                                                                                                                       | If you are unable to disable noop or encounter an error when disabling noop, open a Support ticket referencing S0003, and provide any errors output when attempting to change the setting.                                                                |
| S0004        | Determines if the Puppet Server status endpoint is returning any errors.                                  | Execute `puppet infrastructure status`.  Which ever service returns in a state that is not running, examine the logging for that service to indicate the fault.                                                                                                                                                                                                                                                                                         | Open a Support ticket referencing S0004, provide the output of `puppet infrastructure status` and [any service logs associated with the errors](https://puppet.com/docs/pe/latest/what_gets_installed_and_where.html#log_files_installed).                                                                                    |
| S0005        | Determines if certificate authority (CA) cert expires in the next 90 days.                                         | Install the [puppetlabs-ca_extend](https://forge.puppet.com/modules/puppetlabs/ca_extend) module and follow steps to extend the CA cert.                                                                                                                                                                                                                                                                                                                                             | Open a Support ticket referencing S0005 and provide [support script](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#pe_support_script) output from the primary server, and any errors encountered when using the ca_extend module.                                                                |
| S0006        | Determines if Puppet metrics collector is enabled and collecting metrics.                     |Metrics collector is a tool that lets you monitor a PE installation. If it is not enabled, [enable it.](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#puppet_metrics_collector)  | If you have issues enabling metrics, open a ticket referencing S0006 and provide the output of the [support script.](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#pe_support_script)                                   |
| S0007        | Determines if there is at least 20% disk free on the PostgreSQL data partition.           | Determines if growth is slow and expected within the TTL of your data. If there's an unexpected increase, use this article to [troubleshoot PuppetDB](https://support.puppet.com/hc/en-us/articles/360056219974)                                                                                                                                                                                                 | If your Puppet Practitioner is unable to find a cause for the growth and the suggested KB does not help, open a Support ticket referencing S0007 and provide details about large files and folders, rate of growth, and a full [support script](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#pe_support_script) from the affected node.                                                                          |
| S0008        | Determines if there is at least 20% disk free on the codedir data partition.        | This can indicate you are deploying more code from the code repo than there is space for on the infrastructure components, or that something else is consuming space on this partition. Run `puppet config print codedir`, check that codedir partition indicated has enough capacity for the code being deployed, and check that no other outside files are consuming this data mount.                                                                                    |                                                                                                                                                                                                                                       |
| S0009        | Determines if pe-puppetserver service is running and enabled on relevant components. | Checks that the service can be started and enabled by running `puppet resource service pe-puppetserver ensure=running`, examines `/var/log/puppetlabs/puppetserver/puppetserver.log` for failures.                                                                                                                                                                                                                                                                                               | If you are unable to explain the service outage from the logging, or are unable to start the service, open a Support ticket referencing S0009 and provide the `/var/log/puppetlabs/puppetserver/puppetserver.log`, showing an unsuccessful startup.                                                                              |
| S0010        | Determines if pe-puppetdb service is running and enabled on relevant components.    | Checks that the service can be started and enabled by running `puppet resource service pe-pupeptdb ensure=running`, examines `/var/log/puppetlabs/puppetdb/puppetdb.log` for failures.                                                                                                                                                                                                                                                                                                         | If you are unable to explain the service outage from the logging, or are unable to start the service, open a Support ticket referencing S0010 and provide the `/var/log/puppetlabs/puppetdb/puppetdb.log` log, showing an unsuccessful startup.                                                                                      |
| S0011        | Determines if pe-postgres service is running and enabled on relevant components.    | Checks that the service can be started and enabled by running `puppet resource service pe-postgres ensure=running`, examines `/var/log/puppetlabs/postgresql/<postgresversion>/postgresql-<today>.log` for failures.                                                                                                                                                                                                                                                                           |If you are unable to explain the service outage from the logging, or are unable to start the service, open a Support ticket referencing S0011 and provide the `/var/log/puppetlabs/postgresql/<postgresversion>/ postgresql-<today>.log` log, showing an unsuccessful startup                                                        |
| S0012        | Determines if Puppet produced a report during the last run interval.                | [Troubleshoot Puppet run failures.](https://puppet.com/docs/pe/latest/run_puppet_on_nodes.html#troubleshooting_puppet_run_failures)                                                                                                                                                                                                                                                                                                                                                                             | Open a Support ticket referencing S0012 and provide the output of `puppet agent -td > debug.log 2>&1`                                                                                                                             |
| S0013        | Determines if the catalog was successfully applied during the last  Puppet run.              | [Troubleshoot Puppet run failures.](https://puppet.com/docs/pe/latest/run_puppet_on_nodes.html#troubleshooting_puppet_run_failures)                                                                                                                                                                                                                                                                                                                                                                            | Open a Support ticket referencing S0013 and provide the output of `puppet agent -td > debug.log 2>&1`                                                                                                                            |
| S0014        | Determines if anything in the command queue is older than a Puppet run interval.         | This can indicate that the PuppetDB performance is inadequate for incoming requests. Review PuppetDB performance. [Use metrics to pinpoint the issue.](https://support.puppet.com/hc/en-us/articles/231751308)  | If your are unable to determine the reason from the metrics, open a Support ticket referencing `S0014` and provide the output of the [support script.](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#pe_support_script) and the findings from your analysis of the metrics |
| S0015        | Determines if the infrastructure agent host certificate is expiring in the next 90 days.                                     | Puppet Enterprise has built in functionalilty to regenerate infrastructure certificates, see the following [documentation](https://puppet.com/docs/pe/2021.5/regenerate_certificates.html#regenerate_certificates)  |  If the documented steps fail to resolve your issue, open a support ticket referencing S0015 and provide the error message received when running the steps.                                                                               | 
| S0016        | Determines if there are any `OutOfMemory` errors in the `puppetserver` log. | [Increase the Java heap size for that service.](https://support.puppet.com/hc/en-us/articles/360015511413) | Open a Support ticket referencing S0016 and provide [puppet metrics](https://support.puppet.com/hc/en-us/articles/231751308), `/var/log/puppetlabs/puppetserver/puppetserver.log`, and the output of `puppet infra tune`.|
| S0017        | Determines if there are any `OutOfMemory` errors in the `puppetdb` log. | [Increase the Java heap size for that service.](https://support.puppet.com/hc/en-us/articles/360015511413) | Open a Support ticket referencing S0017 and provide [puppet metrics](https://support.puppet.com/hc/en-us/articles/231751308), `/var/log/puppetlabs/puppetdb/puppetdb.log`, and the output of `puppet infra tune`.|
| S0018        | Determines if there are any `OutOfMemory` errors in the `orchestrator` log. | [Increase the Java heap size for that service.](https://support.puppet.com/hc/en-us/articles/360015511413) | Open a Support ticket referencing S0018 and provide [puppet metrics](https://support.puppet.com/hc/en-us/articles/231751308), `/var/log/puppetlabs/orchestration-services/orchestration-services.log`, and output of `puppet infra tune`.|
|S0019|Determines if there are sufficent jRubies available to serve agents.| Insufficient jRuby availability results in queued puppet agents and overall poor system performance. There can be many causes: [Insufficient server tuning for load](https://support.puppet.com/hc/en-us/articles/360013148854), [a thundering herd](https://support.puppet.com/hc/en-us/articles/215729277), and [insufficient system resources for scale.](https://puppet.com/docs/pe/latest/hardware_requirements.html#hardware_requirements) | If self-sevice fails to resolve the issue, open a ticket referencing S0019 and provide a description of actions so far and the output of the [support script.](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#pe_support_script)
| S0020        | Determines if the Console status api reports all services as running |  Determine which service caused the failure [Service Request Format](https://www.puppet.com/docs/pe/2023.4/status_api_json_endpoints#get_status_v1_services-get-status-v1-services-request-format), go to the [logging] (https://www.puppet.com/docs/pe/2023.4/what_gets_installed_and_where.html?_ga=2.219585753.1594518485.1698057844-280774152.1694007045&_gl=1*xeui3a*_ga*MjgwNzc0MTUyLjE2OTQwMDcwNDU.*_ga_7PSYLBBJPT*MTY5ODMyNzY5MS41Ny4xLjE2OTgzMjgyOTIuMTEuMC4w#log_files_installed) of that service and look for related error messages| Open a Support ticket referencing S0020, please provide the name of the service that failed, time of failure, error messages and provide a copy of the [Support Script](https://www.puppet.com/docs/pe/2023.4/getting_support_for_pe.html#pe_support_script-running-support-script) from your primary. |
|S0021|Determines if free memory is less than 10%.| Ensure your system hardware availability matches the [recommended configuration](https://puppet.com/docs/pe/latest/hardware_requirements.html#hardware_requirements), note this assumes no third-party software using significant resources, adapt requirements accordingly for third-party requirements. Examine metrics from the server and determine if the memory issue is persistent | If you have issues with memory utilization in Puppet Enterprise that can not be explained, open a Support ticket, referencing S0021 and provide the output of the [support script](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#pe_support_script)
| S0022        | Determines if there is a valid Puppet Enterprise license in place at `/etc/puppetlabs/license.key` on the primary server which is not expiring in the next 90 days.                | [Get help with Puppet Enterprise license issues](https://support.puppet.com/hc/en-us/articles/360017933313)| Open a Support ticket referencing S0022 and provide the output of the following commands `ls -la /etc/puppetlabs/license.key` and `cat /etc/puppetlabs/license.key`. |
| S0023        | Determines if certificate authority CRL expires in the next 90 days.                                         |     The solution is to reissue a new CRL from the Puppet CA, note this will also remove any revoked certificates. To do this  follow the instructions in [this module](https://forge.puppet.com/modules/m0dular/crl_truncate)                                                                             | Open a Support ticket referencing S0023 and provide [support script](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#pe_support_script) output from the primary server, and errors or output collected from the resolution steps                                                                |
| S0024        | Determines if there are files in the puppetdb discard directory newer than 1 week old                       |   Recent files indicate an issue that causes PuppetDB to reject incoming data. Investigate Puppetdb logs at the time the data was rejected to find a cause,                                                   | If you are unable to determine a reason for the rejections from logging, Open a Support ticket referencing S0024 and provide a copy of the PuppetDB log for the time in question, along with a sample of the most recent file in the following directory `/opt/puppetlabs/server/data/puppetdb/stockpile/discard/`|
| S0025        | Determines if the host copy of the CRL expires in the next 90 days.                                         | If the Output of S0023 on the primary server is also false use the resolution steps in S0023. If S0023 on the Primary is True, follow [this article](https://support.puppet.com/hc/en-us/articles/7631166251415)  | Open a Support ticket referencing S0025 and provide any errors you received in following the resolution steps                                                                                   |                                                               |
| S0026        | Determines if pe-puppetserver JVM Heap-Memory is set to an inefficient value. | Due to an odditity in how JVM memory is utilised, most applications are unable to consume heap memory between ~31GB and ~48GB as such is if you have heap memory set within this value, you should reduce it to more efficiently allocate server resources. To set heap refer to [Increase the Java heap size for this service.](https://support.puppet.com/hc/en-us/articles/360015511413)                                                                                                                                                                                                                                                                                             |                                                                             |
| S0027        | Determines if if pe-puppetdb JVM Heap-Memory is set to an inefficient value. | Due to an odditity in how JVM memory is utilised, most applications are unable to consume heap memory between ~31GB and ~48GB as such is if you have heap memory set within this value, you should reduce it to more efficiently allocate server resources. To set heap refer to [Increase the Java heap size for this service.](https://support.puppet.com/hc/en-us/articles/360015511413)      |                                                                           |
| S0029        | Determines if number of current connections to Postgresql DB is approaching 90% of the `max_connections` defined.  | First determine the need to increase connections, evaluate if this message appears on every puppet run, or if idle connections from recent component restarts may be to blame. If persistent, impact is minimal unless you need to add more components such as Compilers or Replicas, if you plan to increase the number of components on your system, increase max_connections value. To increase the maximum number of connections in postgres, adjust `puppet_enterprise::profile::database::max_connections`. Consider also increasing `shared_buffers` if that is the case as each connection consumes RAM.                                                                                                                  | Should you be unable to determine the reason for a recent increase in connection use, or are having issue upping the number of connections available, open a Support ticket referencing S0029 and provide the current and future value for `puppet_enterprise::profile::database::max_connections` and we will assist. 
| S0030        | Determines when infrastructure components have the setting `use_cached_catalog` set to true.                        | Don't configure use_cached_catalog on PE infrastructure nodes. It prevents the management of key infrastructure settings. Disable this setting on all infrastructure components. [See our documentation for more information](https://puppet.com/docs/puppet/latest/configuration.html#use-cached-catalog)                                                                                                                                                                                                                    | If you encounter errors after disabling use_cached_catalog, open a Support ticket referencing S0030 and provide the errors.
| S0031        | Determines if old PE agent packages exist on the primary server. | [Remove the old PE agent packages.](https://support.puppet.com/hc/en-us/articles/4405333422103) |
| S0033        | Determines if Hiera 5 is in use. | Upgrading to Hiera 5 [offers some major advantages](https://puppet.com/docs/puppet/latest/hiera_migrate)                                                                                                                          | If you're having issues upgrading to Hiera 5 or if your global Hiera configuration file was erroneously modified, open a Support ticket referencing S0033. Provide your global Hiera configuration file `puppet config print hiera_config`; the default location is `/etc/puppetlabs/puppet/hiera.yaml`.
| S0034        | Determines if your PE deployment has not been upgraded in the last year.                   |  [Upgrade your PE instance.](https://puppet.com/docs/pe/latest/upgrading_pe.html)                                                                                                                | If you have issues during a Puppet Upgrade, open a ticket and provide your current version and the version you would like to upgrade to and state any problems, providing any logging that is helpful. |
| S0035        | Determines if ``puppet module list`` is returning any warnings | If S0035 returns ``false``, i.e., warnings are present, you should run `puppet module list --debug` and resolve the issues shown.  The Puppetfile does NOT include Forge module dependency resolution. You must make sure that you have every module needed for all of your specified modules to run.Please refer to [Managing environment content with a Puppetfile](https://puppet.com/docs/pe/latest/puppetfile.html) for more info on Puppetfile and refer to the specific module page on the forge for further information on specific dependancies                                                                     | If you are unable to remove all the warnings, then please refer to [Get help for supported modules]( https://support.puppet.com/hc/en-us/articles/226771168) and raise a support request
| S0036        | Determines if `max-queued-requests` is set above 150.                       | [The maximum value for `jruby_puppet_max_queued_requests` is 150](https://support.puppet.com/hc/en-us/articles/115003769433)                                                                                    | If you are unable to change the value of `jruby_puppet_max_queued_requests` or encounter an error when changing it, open a Support ticket referencing S0036 and provide any errors output when attempting to change the setting.
| S0038        | Determines whether the number of environments within `$codedir/environments` is less than 100 | Having a large number of code environments can negatively affect Puppet Server performance. [See the Configuring Puppet Server documentation for more information.](https://puppet.com/docs/pe/latest/config_puppetserver.html#configuring_and_tuning_puppet_server) You should examine if you need them all, any unused environments should be removed. If all are required you can ignore this warning.                    | 
| S0039        | Determines if Puppets Server has reached its `queue-limit-hit-rate`,and is sending messages to agents. | [Check the  max-queued-requests article for more information.](https://support.puppet.com/hc/en-us/articles/115003769433)                      | If the article is unable to solve your issue, open a Support ticket referencing S0039, indicating the investigation so far, and any issues you encountered, then provide the [support script](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#pe_support_script) output from the primary server.
| S0040        | Determines if PE is collecting system metrics.                    | If system metrics are not collected by default, the sysstat package is not installed on the impacted PE infrastructure component. Install the package and set the parameter `puppet_enterprise::enable_system_metrics_collection` to true. [See the documentation.](https://puppet.com/docs/pe/latest/getting_support_for_pe.html#puppet_metrics_collector)                                                                   | After system metrics are configured, you do not see any files in `/var/log/sa` or if the `/var/log/sa` directory does not exist, open a Support ticket.                                                 |
| S0041        | Determines if the pxp broker on a compiler  has an established connection to another pxp broker  | To resolve a connection issue from a compiler to a pcp broker examine the following log `/var/log/puppetlabs/puppetserver/pcp-broker.log` for an explanation, Compilers should be attempting to make a connection to port 8143 on the primary server, ssl can not be terminated on a network appliance and must passthrough directly to the primary server. Ensure the connnection attempt is not to another compiler in the pool            | If unable to make a connection to a broker, raise a ticket with the support team quoting S0041 and attaching the file `/var/log/puppetlabs/puppetserver/pcp-broker.log` along with the conclusions of your investigation so far            |
| S0042        | Determines if the pxp-agent has an established connection to a pxp broker                   | Ensure the pxp-agent service is running. Check S0002 can make that determination. if running check  `/var/log/puppetlabs/pxp-agent/pxp-agent.log` (on *nix) or `C:/ProgramData/PuppetLabs/pxp-agent/var/log/pxp-agent.log` (on Windows), for connection issues, first ensuring the agent is connecting to the proper endpoint, for example, a compiler and not the primary. This fact can also be used as a target filter for running tasks, ensuring time is not wasted sending instructions to agents not connected to a broker                   | If unable to make a connection to a broker, raise a ticket with the support team quoting S0042 and attaching the file  `/var/log/puppetlabs/pxp-agent/pxp-agent.log` (on *nix) or `C:/ProgramData/PuppetLabs/pxp-agent/var/log/pxp-agent.log` (on Windows), along with the conclusions of your investigation so far             |
| S0043        | Determines if there are nodes with Puppet agent versions ahead of the primary server                    | Agent nodes should not be running Puppet agent versions ahead of infrastructure nodes. Instead consider upgrading PE so that PE package management contains the desired Puppet agent version. See the [upgrading PE](https://puppet.com/docs/pe/latest/upgrading_pe.html) and [upgrading agents](https://puppet.com/docs/latest/upgrading_agents.html) documentation for more information.                 | If you are unable to determine why the indicator is evaluating to `false` or have questions about Puppet agent versions, open a support ticket and reference S0043.            |
| S0044        | Determines if Puppet Servers are using the the PE classifier for the node data plugin (node terminus)                    | Due to performance optimizations, it is recommended to use the PE classifier plugin instead of external node classifier (ENC) scripts or applications. See the [node_terminus configuration setting documentation](https://www.puppet.com/docs/puppet/7/configuration.html#node-terminus) for more information.                 | If you have additional questions about the node_terminus configuration setting, open a support ticket and reference S0044.            |
| S0045        | Determines if Puppet Servers are configured with an excessive number of JRubies. | Because each JRuby instance consumes additional memory, having too many can reduce the amount of heap space available to Puppet server and cause excessive garbage collections.  While it is possible to increase the heap along with the number of JRubies, we have observered diminishing returns with more than 12 JRubies and therefore recommend an upper limit of 12. We also recommend allocating between 1 - 2gb of heap memory for each JRuby. |  If you would like to measure the effects of changing JRubies and heap settings, use the [Puppet Operational Dashboards module](https://forge.puppet.com/modules/puppetlabs/puppet_operational_dashboards/readme) to configure a metrics stack and Grafana dashboards for viewing the metrics.  If you still have performance issues or further questions, open a support ticket and reference S0045. |

| AS001        | Determines if the agent host certificate is expiring in the next 90 days.                                     | Puppet Enterprise has a plan built into extend agent certificates. Use a puppet query to find expiring host certificates and pass the node ID to this plan:   `puppet plan run enterprise_tasks::agent_cert_regen agent=$(puppet query 'inventory[certname] { facts.agent_status_check.AS001 = false }' \| jq  -r '.[].certname' \|  paste -sd, -) master=$(puppet config print certname)`  |  If the plan fails to run, open a support ticket referencing AS001 and provide the error message received when running the plan.                                                                               | 
| AS002        | Determines if the pxp-agent has an established connection to a pxp broker                                     | Ensure the pxp-agent service is running, if running check `/var/log/puppetlabs/pxp-agent/pxp-agent.log` (on *nix) or `C:/ProgramData/PuppetLabs/pxp-agent/var/log/pxp-agent.log` (on Windows) — Contains the for connection issues, first ensuring the agent is connecting to the proper endpoint, for example, a compiler and not the primary. This fact can also be used as a target filter for running tasks, ensuring time is not wasted sending instructions to agents not connected to a broker| If unable to make a connection to a broker, raise a ticket with the support team quoting AS002 and attaching the file  `/var/log/puppetlabs/pxp-agent/pxp-agent.log` (on *nix) or `C:/ProgramData/PuppetLabs/pxp-agent/var/log/pxp-agent.log` (on Windows) along with the conclusions of your investigation so far                                                                            | 
| AS003        | Determines the certname configuration parameter is incorrectly set outside of the [main] section of the puppet.conf file.                                 | The Puppet documentation states clearly certname should always be placed solely in the [main] section to prevent unforseen issues with the operation of the puppet agent https://puppet.com/docs/puppet/7/configuration.html#certname | If unable to determine why the indicator is being raised. Open a ticket with the support team quoting AS003 and attaching the file  `puppet.conf`  along with the conclusions of your investigation so far .                                                                           | 
| AS004        | Determines if the host copy of the CRL expires in the next 90 days.                                         | If the Output of S0023 on the primary server is also false use the resolution steps in S0023. If S0023 on the Primary is True, follow [this article](https://support.puppet.com/hc/en-us/articles/7631166251415)  | Open a Support ticket referencing AS004 and provide any errors you recieved in following the resolution steps                                                                                   |

## How to report an issue or contribute to the module

 If you have a reproducible bug, you can [open an issue directly in the GitHub issues page of the module](https://github.com/h0tw1r3/h0tw1r3-puppet_status_check/issues). We also welcome PR contributions to improve the module. [Please see further details about contributing](https://puppet.com/docs/puppet/latest/contributing.html#contributing_changes_to_module_repositories).
