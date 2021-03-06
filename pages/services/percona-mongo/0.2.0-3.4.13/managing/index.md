---
layout: layout.pug
navigationTitle:  Managing
title: Managing
menuWeight: 60
excerpt:
featureMaturity:
enterprise: false
---

# Updating Configuration

You can make changes to the service after it has been launched. Configuration management is handled by the scheduler process, which in turn handles deploying Percona-Mongo itself.

After making a change, the scheduler will be restarted, and it will automatically deploy any detected changes to the service, one node at a time. For example, a given change will first be applied to `mongo-rs-0`, then `mongo-rs-1`, and so on.

Nodes are configured with a "Readiness check" to ensure that the underlying service appears to be in a healthy state before continuing with applying a given change to the next node in the sequence. However, this basic check is not foolproof and reasonable care should be taken to ensure that a given configuration change will not negatively affect the behavior of the service.

Some changes, such as decreasing the number of nodes or changing volume requirements, are not supported after initial deployment. See [Limitations](#limitations).

<!-- THIS CONTENT DUPLICATES THE DC/OS OPERATION GUIDE -->

The instructions below describe how to update the configuration for a running DC/OS service.

### Enterprise DC/OS 1.10

Enterprise DC/OS 1.10 introduces a convenient command line option that allows for easier updates to a service's configuration, as well as allowing users to inspect the status of an update, to pause and resume updates, and to restart or complete steps if necessary.

#### Prerequisites

+ Enterprise DC/OS 1.10 or newer.
+ Service with a version greater than 2.0.0-x.
+ [The DC/OS CLI](https://docs.mesosphere.com/latest/cli/install/) installed and available.
+ The service's subcommand available and installed on your local machine.
  + You can install just the subcommand CLI by running `dcos package install --cli percona-mongo`.
  + If you are running an older version of the subcommand CLI that doesn't have the `update` command, uninstall and reinstall your CLI.
    ```shell
    dcos package uninstall --cli percona-mongo
    dcos package install --cli percona-mongo
    ```

#### Preparing configuration

If you installed this service with Enterprise DC/OS 1.10, you can fetch the full configuration of a service (including any default values that were applied during installation). For example:

```shell
dcos percona-mongo describe > options.json
```

Make any configuration changes to this `options.json` file.

If you installed this service with a prior version of DC/OS, this configuration will not have been persisted by the the DC/OS package manager. You can instead use the `options.json` file that was used when [installing the service](#initial-service-configuration).

<strong>Note:</strong> You need to specify all configuration values in the `options.json` file when performing a configuration update. Any unspecified values will be reverted to the default values specified by the DC/OS service. See the "Recreating `options.json`" section below for information on recovering these values.

##### Recreating `options.json` (optional)

If the `options.json` from when the service was last installed or updated is not available, you will need to manually recreate it using the following steps.

First, we'll fetch the default application's environment, current application's environment, and the actual template that maps config values to the environment:

1. Ensure you have [jq](https://stedolan.github.io/jq/) installed.
1. Set the service name that you're using, for example:
```shell
SERVICE_NAME=percona-mongo
```
1. Get the version of the package that is currently installed:
```shell
PACKAGE_VERSION=$(dcos package list | grep $SERVICE_NAME | awk '{print $2}')
```
1. Then fetch and save the environment variables that have been set for the service:
```shell
dcos marathon app show $SERVICE_NAME | jq .env > current_env.json
```
1. To identify those values that are custom, we'll get the default environment variables for this version of the service:
```shell
dcos package describe --package-version=$PACKAGE_VERSION --render --app $SERVICE_NAME | jq .env > default_env.json
```
1. We'll also get the entire application template:
```shell
dcos package describe $SERVICE_NAME --app > marathon.json.mustache
```

Now that you have these files, we'll attempt to recreate the `options.json`.

1. Use JQ and `diff` to compare the two:
```shell
diff <(jq -S . default_env.json) <(jq -S . current_env.json)
```
1. Now compare these values to the values contained in the `env` section in application template:
```shell
less marathon.json.mustache
```
1. Use the variable names (e.g. `{{service.name}}`) to create a new `options.json` file as described in [Initial service configuration](#initial-service-configuration).

#### Starting the update

Once you are ready to begin, initiate an update using the DC/OS CLI, passing in the updated `options.json` file:

```shell
dcos percona-mongo update start --options=options.json
```

You will receive an acknowledgement message and the DC/OS package manager will restart the Scheduler in Marathon.

See [Advanced update actions](#advanced-update-actions) for commands you can use to inspect and manipulate an update after it has started.

### Open Source DC/OS

If you do not have Enterprise DC/OS 1.10 or later, the CLI commands above are not available. For Open Source DC/OS of any version you can perform changes from the DC/OS GUI.

<!-- END DUPLICATE BLOCK -->

To make configuration changes via scheduler environment updates, perform the following steps:
1. Visit <dcos-url> to access the DC/OS web interface.
1. Navigate to `Services` and click on the service to be configured (default _`percona-mongo`_).
1. Click `Edit` in the upper right.
1. Navigate to `Environment` (or `Environment variables`) and search for the option to be updated.
1. Update the option value and click `Review and run` (or `Deploy changes`).
1. The Scheduler process will be restarted with the new configuration and will validate any detected changes.
1. If the detected changes pass validation, the relaunched Scheduler will deploy the changes by sequentially relaunching affected tasks as described above.

To see a full listing of available options, run `dcos package describe --config percona-mongo` in the CLI, or browse the DC/OS Percona Server for MongoDB Service install dialog in the DC/OS Dashboard.

<a name="adding-a-node"></a>
### Adding a Node

The service deploys 3 nodes by default, as 3 nodes is the minimum node requirement for a Highly-Available [MongoDB Replica Set](https://docs.mongodb.com/manual/replication/). **Note: Only 1 *(not recommended)*, 3, 5 or 7 nodes is recommended and supported.**

You can customize this value at initial deployment or after the cluster is already running. Shrinking the count is not supported.

Modify the `NODE_COUNT` environment variable to update the node count. If you decrease this value, the scheduler will prevent the configuration change until it is reverted back to its original value or larger.

<a name="resizing-a-node"></a>
### Resizing a Node

The CPU and Memory requirements of each node can be increased or decreased as follows:
- CPU (1.0 = 1 core): `NODE_CPUS`
- Memory (in MB): `NODE_MEM`

**Note:** Volume requirements (type and/or size) cannot be changed after initial deployment.

<a name="updating-placement-constraints"></a>
### Updating Placement Constraints

Placement constraints may be updated after initial deployment using the following procedure. See [Service Settings](#service-settings) above for more information on placement constraints.

Let's say we have the following deployment of our nodes

- Placement constraint of: `hostname:LIKE:10.0.10.3|10.0.10.8|10.0.10.26|10.0.10.28|10.0.10.84`
- Tasks:
```
10.0.10.3: mongo-rs-0
10.0.10.8: mongo-rs-1
10.0.10.26: mongo-rs-2
10.0.10.28: empty
10.0.10.84: empty
```

`10.0.10.8` is being decommissioned and we should move away from it. Steps:

1. Remove the decommissioned IP and add a new IP to the placement rule whitelist by editing `NODE_PLACEMENT`:

	```
	hostname:LIKE:10.0.10.3|10.0.10.26|10.0.10.28|10.0.10.84|10.0.10.123
	```
    1. Redeploy `mongo-rs-1` from the decommissioned node to somewhere within the new whitelist: `dcos percona-mongo pod replace mongo-rs-1`
1. Wait for `mongo-rs-1` to be up and healthy before continuing with any other replacement operations.

<a name="restarting-a-node"></a>
## Restarting a Node

This operation will restart a node, while keeping it at its current location and with its current persistent volume data. This may be thought of as similar to restarting a system process, but it also deletes any data that is not on a persistent volume.

1. Run `dcos percona-mongo pod restart mongo-rs-<NUM>`, e.g. `mongo-rs-2`.

<a name="replacing-a-node"></a>
## Replacing a Node

This operation will move a node to a new system and will discard the persistent volumes at the prior system to be rebuilt at the new system. Perform this operation if a given system is about to be offlined or has already been offlined.

**Note:** Nodes are not moved automatically. You must perform the following steps manually to move nodes to new systems. You can automate node replacement according to your own preferences.

1. For data safety, ensure there is at least one healthy node in the replica set and a recent successful backup of MongoDB data! Ensure there is a ["majority" in the MongoDB Replica Set](https://docs.mongodb.com/manual/reference/write-concern/#writeconcern._dq_majority_dq_) if you do not want the replica set to become read-only during the node replacement!
1. Connect to the node and check if the node is the [Replica Set Primary](https://docs.mongodb.com/manual/core/replica-set-primary/). If the *'rs.isMaster().ismaster'* command returns *'true'*, the node is the PRIMARY member.
    ```shell
    $ mongo mongodb://clusteruseradmin:clusteruseradminpassword@mongo-rs-<NUM>-mongod.percona-mongo.autoip.dcos.thisdcos.directory:27017/admin?replicaSet=rs
    > rs.isMaster().ismaster
    true
    ```
1. If the node is the PRIMARY member, run a [Replica Set Step Down](https://docs.mongodb.com/manual/reference/method/rs.stepDown/). Skip this step if you received *'false'* from the last step.
    ```shell
    $ mongo mongodb://clusteruseradmin:clusteruseradminpassword@mongo-rs-<NUM>-mongod.percona-mongo.autoip.dcos.thisdcos.directory:27017/admin?replicaSet=rs
    > rs.stepDown()
    2018-04-03T13:58:29.166+0200 E QUERY    [thread1] Error: error doing query: failed: network error while attempting to run command 'replSetStepDown' on host '10.2.3.1:27017'  :
    DB.prototype.runCommand@src/mongo/shell/db.js:132:1
    DB.prototype.adminCommand@src/mongo/shell/db.js:150:16
    rs.stepDown@src/mongo/shell/utils.js:1274:12
    @(shell):1:1
    2018-04-03T13:58:29.167+0200 I NETWORK  [thread1] trying reconnect to 10.2.3.1:27017 (10.2.3.1) failed
    2018-04-03T13:58:29.168+0200 I NETWORK  [thread1] reconnect 10.2.3.1:27017 (10.2.3.1) ok
    test1:SECONDARY>
    ```
1. Run `dcos percona-mongo pod replace mongo-rs-<NUM>` to halt the current instance with id `<NUM>` (if still running) and launch a new instance elsewhere.

For example, let's say `mongo-rs-2`'s host system has died and `mongo-rs-2` needs to be moved.

1. Connect to the node and check if the node is the [Replica Set Primary](https://docs.mongodb.com/manual/core/replica-set-primary/). If the *'rs.isMaster().ismaster'* command returns *'true'*, the node is the PRIMARY member.
    ```shell
    $ mongo mongodb://clusteruseradmin:clusteruseradminpassword@mongo-rs-2-mongod.percona-mongo.autoip.dcos.thisdcos.directory:27017/admin?replicaSet=rs
    > rs.isMaster().ismaster
    true
    ```
1. If the node is the PRIMARY member, run a [Replica Set Step Down](https://docs.mongodb.com/manual/reference/method/rs.stepDown/). Skip this step if you received *
'false'* from the last step.
    ```shell
    $ mongo mongodb://clusteruseradmin:clusteruseradminpassword@mongo-rs-<NUM>-mongod.percona-mongo.autoip.dcos.thisdcos.directory:27017/admin?replicaSet=rs
    > rs.stepDown()
    2018-04-03T13:58:29.166+0200 E QUERY    [thread1] Error: error doing query: failed: network error while attempting to run command 'replSetStepDown' on host '10.2.3.1:27017'  :
    DB.prototype.runCommand@src/mongo/shell/db.js:132:1
    DB.prototype.adminCommand@src/mongo/shell/db.js:150:16
    rs.stepDown@src/mongo/shell/utils.js:1274:12
    @(shell):1:1
    2018-04-03T13:58:29.167+0200 I NETWORK  [thread1] trying reconnect to 10.2.3.1:27017 (10.2.3.1) failed
    2018-04-03T13:58:29.168+0200 I NETWORK  [thread1] reconnect 10.2.3.1:27017 (10.2.3.1) ok
    test1:SECONDARY>
    ```
1. Now that the node has been decommissioned, start `mongo-rs-2` at a new location in the cluster.
    ```shell
    dcos percona-mongo pod replace mongo-rs-2
    ```

<!-- THIS CONTENT DUPLICATES THE DC/OS OPERATION GUIDE -->

<a name="upgrading"></a>
## Upgrading Service Version

The instructions below show how to safely update one version of percona-mongo to the next.

##### Viewing available versions

The `update package-versions` command allows you to view the versions of a service that you can upgrade or downgrade to. These are specified by the service maintainer and depend on the semantics of the service (i.e. whether or not upgrades are reversal).

For example, run:

```shell
dcos percona-mongo update package-versions
```

## Upgrading or downgrading a service

1. Before updating the service itself, update its CLI subcommand to the new version:
```shell
dcos package uninstall --cli percona-mongo
dcos package install --cli percona-mongo -package-version="0.2.0-3.4.13"
```
1. Once the CLI subcommand has been updated, call the update start command, passing in the version. For example, to update DC/OS Percona Server for MongoDB Service to version `0.2.0-3.4.13`:
```shell
dcos percona-mongo update start --package-version="0.2.0-3.4.13"
```

If you are missing mandatory configuration parameters, the `update` command will return an error. To supply missing values, you can also provide an `options.json` file (see [Updating configuration](#updating-configuration)):
```shell
dcos percona-mongo update start --options=options.json --package-version="0.2.0-3.4.13"
```

See [Advanced update actions](#advanced-update-actions) for commands you can use to inspect and manipulate an update after it has started.

<!-- END DUPLICATE BLOCK -->

## Advanced update actions

<!-- THIS CONTENT DUPLICATES THE DC/OS OPERATION GUIDE -->

The following sections describe advanced commands that be used to interact with an update in progress.

### Monitoring the update

Once the Scheduler has been restarted, it will begin a new deployment plan as individual pods are restarted with the new configuration. Depending on the high availability characteristics of the service being updated, you may experience a service disruption.

You can query the status of the update as follows:

```shell
dcos percona-mongo update status
```

If the Scheduler is still restarting, DC/OS will not be able to route to it and this command will return an error message. Wait a short while and try again. You can also go to the Services tab of the DC/OS GUI to check the status of the restart.

### Pause

To pause an ongoing update, issue a pause command:

```shell
dcos percona-mongo update pause
```

You will receive an error message if the plan has already completed or has been paused. Once completed, the plan will enter the `WAITING` state.

### Resume

If a plan is in a `WAITING` state, as a result of being paused or reaching a breakpoint that requires manual operator verification, you can use the `resume` command to continue the plan:

```shell
dcos percona-mongo update resume
```

You will receive an error message if you attempt to `resume` a plan that is already in progress or has already completed.

### Force Complete

In order to manually "complete" a step (such that the Scheduler stops attempting to launch a task), you can issue a `force-complete` command. This will instruct to Scheduler to mark a specific step within a phase as complete. You need to specify both the phase and the step, for example:

```shell
dcos percona-mongo update force-complete service-phase service-0:[node]
```

### Force Restart

Similar to force complete, you can also force a restart. This can either be done for an entire plan, a phase, or just for a specific step.

To restart the entire plan:
```shell
dcos percona-mongo update force-restart
```

Or for all steps in a single phase:
```shell
dcos percona-mongo update force-restart service-phase
```

Or for a specific step within a specific phase:
```shell
dcos percona-mongo update force-restart service-phase service-0:[node]
```

<!-- END DUPLICATE BLOCK -->
