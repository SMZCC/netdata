<!--
---
title: "Agent claiming"
description: "Agent claiming allows a Netdata Agent, running on a distributed node, to securely connect to Netdata Cloud. A Space's administrator creates a claiming token, which is used to add an Agent to their Space via the Agent-Cloud link."
date: 2020-04-15
custom_edit_url: https://github.com/netdata/netdata/edit/master/claim/README.md
---
-->

# Agent claiming

Agent claiming allows a Netdata Agent, running on a distributed node, to securely connect to Netdata Cloud. A Space's
administrator creates a **claiming token**, which is used to add an Agent to their Space via the [Agent-Cloud link
(ACLK)](/aclk/README.md).

Claiming nodes is a security feature in Netdata Cloud. Through the process of claiming, you demonstrate in a few ways
that you have administrative access to that node and the configuration settings for its Agent. By logging into the node,
you prove you have access, and by using the claiming script or the Netdata command line, you prove you have write access
and administrative privileges.

Only the administrators of a Space in Netdata Cloud can view the claiming token and accompanying script generated by
Netdata Cloud.

> The claiming process ensures no third party can add your node, and then view your node's metrics, in a Cloud account,
> Space, or War Room that you did not authorize.

By claiming a node, you opt-in to sending data from your Agent to Netdata Cloud via the ACLK. This data is encrypted by
TLS while it is in transit. We use the the RSA keypair created during claiming to authenticate the identity of the agent
when it connects to the Cloud. While the data does flow through Netdata Cloud servers on its way from Agents to the
browser, we do not store or log it.

## How to claim a node

You can claim a node during the Cloud onboarding process, or after you created a Space by clicking on the **USER's
Space** dropdown, then **Manage Claimed Nodes**.

> Only the administrators of a Space in Netdata Cloud can view the claiming token and accompanying script generated by
> Netdata Cloud.

To claim a node, copy the script given by Cloud. **You must run this script as the user running the** `netdata`
**service**, which is usually the `netdata` user. You have two options: Switch to the appropriate user yourself, or run
the command using `sudo` to automatically manage permissions.

By switching to the `netdata` user:

```bash
sudo su -s /bin/bash netdata
netdata-claim.sh -token=TOKEN -rooms=ROOM1,ROOM2 -url=https://app.netdata.cloud
```

With `sudo`:

```bash
sudo netdata-claim.sh -token=TOKEN -rooms=ROOM1,ROOM2 -url=https://app.netdata.cloud
```

Hit **Enter**. The script should return `Agent was successfully claimed.`.

> Your node may need up to 60 seconds to connect to Netdata Cloud after finishing the claiming process. Please be
> patient!

If the claiming script returns errors, see the [troubleshooting information](#troubleshooting).

### Claiming through a proxy

A Space's administrator can claim a node through a SOCKS5 or HTTP(S) proxy.

You should first configure the proxy in the `[cloud]` section of `netdata.conf`. The proxy settings you specify here
will also be used to tunnel the ACLK. The default `proxy` setting is `none`.

```ini
[cloud]
    proxy = none
```

The `proxy` setting can take one of the following values:

-   `none`: Do not use a proxy, even if the system configured otherwise.
-   `env`: Try to read proxy settings from set environment variables `http_proxy`/`socks_proxy`.
-   `socks5[h]://[user:pass@]host:ip`: The ACLK and claiming will use the specified SOCKS5 proxy.
-   `http://[user:pass@]host:ip`: The ACLK and claiming will use the specified HTTP(S) proxy.

For example, a SOCKS5 proxy setting may look like the following:

```conf
[cloud]
    proxy = socks5h://203.0.113.0:1080       # With an IP address
    proxy = socks5h://proxy.example.com:1080 # With a URL
```

You can now move on to claiming. Be sure to switch to the `netdata` user or use `sudo` as explained in the [step above](#how-to-claim-a-node).

When you claim with the `netdata-claim.sh` script, add the `-proxy=` parameter and append the same proxy setting you
added to `netdata.conf`.

```bash
netdata-claim.sh -token=MYTOKEN1234567 -rooms=room1,room2 -url=https://app.netdata.cloud -proxy=socks5h://203.0.113.0:1080
```

Hit **Enter**. The script should return `Agent was successfully claimed.`.

> Your node may need up to 60 seconds to connect to Netdata Cloud after finishing the claiming process. Please be
> patient!

If the claiming script returns errors, see the [troubleshooting information](#troubleshooting).

### Troubleshooting

If you're having trouble claiming a node, this may be because the ACLK cannot connect to Cloud.

With the Netdata Agent running, visit `http://127.0.0.1/api/v1/info` in your browser. The returned JSON contains four
keys that will be helpful to diagnose any issues you might be having with the ACLK or claiming process.

```json
	"cloud-enabled"
	"cloud-available"
	"agent-claimed"
	"aclk-available"
```

Use these keys and the information below to troubleshoot the ACLK.

#### cloud-enabled is false

If `cloud-enabled` is `false`, you probably ran the installer with `--disable-cloud` option.

Additionally, check that the `netdata cloud` setting in `netdata.conf` is set to `enable`:

```ini
[general]
    netadata cloud = enable
```

To fix this issue, reinstall Netdata using your [preferred method](/packaging/installer/README.md) and do not add the
`--disable-cloud` option.

#### cloud-available is false

If `cloud-available` is `false` after you verified Cloud is enabled in the previous step, the most likely issue is that
Cloud features failed to build during installation.

If Cloud features fail to build, the installer continues and finishes the process without Cloud functionality as opposed
to failing the installation altogether. We do this to ensure the Agent will always finish installing.

If you can't see an explicit error in the installer's output, you can run the installer with the `--require-cloud`
option. This option causes the installation to fail if Cloud functionality can't be built and enabled, and the
installer's output should give you more error details.

You may see one of the following error messages during installation:

-   Failed to build libmosquitto. The install process will continue, but you will not be able to connect this node to
    Netdata Cloud.
-   Unable to fetch sources for libmosquitto. The install process will continue, but you will not be able to connect
    this node to Netdata Cloud.
-   Failed to build libwebsockets. The install process will continue, but you may not be able to connect this node to
    Netdata Cloud.
-   Unable to fetch sources for libwebsockets. The install process will continue, but you may not be able to connect
    this node to Netdata Cloud.

One common cause of the installer failing to build Cloud features is not having one of the following dependencies on
your system: `cmake` and OpenSSL, including the `devel` package.

You can also look for error messages in `/var/log/netdata/error.log`. Try one of the following two commands to search
for ACLK-related errors.

```bash
less /var/log/netdata/error.log
grep -i ACLK /var/log/netdata/error.log
```

If the installer's output does not help you enable Cloud features, contact us by [creating an issue on
GitHub](https://github.com/netdata/netdata/issues/new?labels=bug%2C+needs+triage%2C+ACLK&template=bug_report.md&title=The+installer+failed+to+prepare+the+required+dependencies+for+Netdata+Cloud+functionality)
with details about your system and relevant output from `error.log`.

#### agent-claimed is false

You must [claim your node](#how-to-claim-a-node).

#### aclk-available is false

If `aclk-available` is `false` and all other keys are `true`, your Agent is having trouble connection to the Cloud
through the ACLK. Please check your system's firewall.

If your Agent needs to use a proxy to access the internet, you must [set up a proxy for
claiming](#claiming-through-a-proxy).

If you are certain firewall and proxy settings are not the issue, you should consult the Agent's `error.log` at
`/var/log/netdata/error.log` and contact us by [creating an issue on
GitHub](https://github.com/netdata/netdata/issues/new?labels=bug%2C+needs+triage%2C+ACLK&template=bug_report.md&title=ACLK-available-is-false)
with details about your system and relevant output from `error.log`.

### Unclaim (remove) an Agent from Netdata Cloud

The best method to remove an Agent from Netdata Cloud is to unclaim it by deleting the `claim.d/` directory in your
Netdata configuration directory.

```bash
cd /etc/netdata   # Replace with your Netdata configuration directory, if not /etc/netdata/
rm -rf claim.d/
```

> You may need to use `sudo` or another method of elevating your privileges.

Once you delete the `claim.d/` directory, the ACLK will not connect to Cloud the next time the Agent starts, and Cloud
will then remove it from the interface.

## Claiming reference

In the sections below, you can find reference material for the claiming script, claiming via the Agent's command line
tool, and details about the files found in `claim.d`.

### Claiming script

A Space's administrator can claim an Agent by directly calling the `netdata-claim.sh` script **as the `netdata` user**
and passing the following arguments:

```sh
-token=TOKEN
    where TOKEN is the Space's claiming token.
-rooms=ROOM1,ROOM2,...
    where ROOMX is the War Room this node should be added to. This list is optional.
-url=URL_BASE
    where URL_BASE is the Netdata Cloud endpoint base URL. By default, this is https://netdata.cloud.
-id=AGENT_ID
    where AGENT_ID is the unique identifier of the Agent. This is the Agent's MACHINE_GUID by default.
-hostname=HOSTNAME
    where HOSTNAME is the result of the hostname command by default.
-proxy=PROXY_URL
    where PROXY_URL is the endpoint of a SOCKS5 proxy.
```

For example, the following command claims an Agent and adds it to rooms `room1` and `room2`:

```sh
netdata-claim.sh -token=MYTOKEN1234567 -rooms=room1,room2
```

You should then update the `netdata` service about the result with `netdatacli`:

```sh
netdatacli reload-claiming-state
```

This reloads the Agent claiming state from disk.

### Netdata Agent command line

If a Netdata Agent is running, the Space's administrator can claim a node using the `netdata` service binary with
additional command line parameters:

```sh
-W "claim -token=TOKEN -rooms=ROOM1,ROOM2"
```

For example:

```sh
/usr/sbin/netdata -D -W "claim -token=MYTOKEN1234567 -rooms=room1,room2"
```

If need be, the user can override the Agent's defaults by providing additional arguments like those described
[here](#claiming-script).

### Claiming directory

Netdata stores the agent claiming-related state in the user configuration directory under `claim.d`, e.g. in
`/etc/netdata/claim.d`. The user can put files in this directory to provide defaults to the `-token` and `-rooms`
arguments. These files should be owned **by the `netdata` user**.

The `claim.d/token` file should contain the claiming-token and the `claim.d/rooms` file should contain the list of 
war-rooms.

The user can also put the Cloud endpoint's full certificate chain in `claim.d/cloud_fullchain.pem` so that the Agent
can trust the endpoint if necessary.

[![analytics](https://www.google-analytics.com/collect?v=1&aip=1&t=pageview&_s=1&ds=github&dr=https%3A%2F%2Fgithub.com%2Fnetdata%2Fnetdata&dl=https%3A%2F%2Fmy-netdata.io%2Fgithub%2Fclaim%2FREADME&_u=MAC~&cid=5792dfd7-8dc4-476b-af31-da2fdb9f93d2&tid=UA-64295674-3)](<>)
