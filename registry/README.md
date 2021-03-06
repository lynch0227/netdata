# Netdata registry

Netdata registry implements the `my-netdata` menu on netdata dashboards.
The `my-netdata` menu lists the netdata servers you have visited.

## Why?

Netdata provides distributed monitoring.

Traditional monitoring solutions centralize all the data to provide unified dashboards across all servers. Before netdata, this was the standard practice. However it has a few issues:

1. due to the resources required, the number of metrics collected is limited.
1. for the same reason, the data collection frequency is not that high, at best it will be once every 10 or 15 seconds, at worst every 5 or 10 mins.
1. the central monitoring solution needs dedicated resources, thus becoming "another bottleneck" in the whole ecosystem. It also requires maintenance, administration, etc.
1. most centralized monitoring solutions are usually only good for presenting *statistics of past performance* (i.e. cannot be used for real-time performance troubleshooting).

Netdata follows a different approach:

1. data collection happens per second
1. thousands of metrics per server are collected
1. data do not leave the server where they are collected
1. netdata servers do not talk to each other
1. your browser connects all the netdata servers

Using netdata, your monitoring infrastructure is embedded on each server, limiting significantly the need of additional resources. Netdata is blazingly fast, very resource efficient and utilizes server resources that already exist and are spare (on each server). This allows **scaling out** the monitoring infrastructure.

However, the netdata approach introduces a few new issues that need to be addressed, one being **the list of netdata we have installed**, i.e. the URLs our netdata servers are listening.

To solve this, netdata utilizes a **central registry**. This registry, together with certain browser features, allow netdata to provide unified cross-server dashboards. For example, when you jump from server to server using the `my-netdata` menu, several session settings (like the currently viewed charts, the current zoom and pan operations on the charts, etc.) are propagated to the new server, so that the new dashboard will come with exactly the same view.

## What is the registry?

The registry keeps track of 3 entities:

1. **machines**: i.e. the netdata installations (a random GUID generated by each netdata the first time it starts; we call this **machine_guid**)

  For each netdata installation (each `machine_guid`) the registry keeps track of the different URLs it is accessed.

1. **persons**: i.e. the web browsers accessing the netdata installations (a random GUID generated by the registry the first time it sees a new web browser; we call this **person_guid**)

  For each person, the registry keeps track of the netdata installations it has accessed and their URLs.

1. **URLs** of netdata installations (as seen by the web browsers)

  For each URL, the registry keeps the URL and nothing more. Each URL is linked to *persons* and *machines*. The only way to find a URL is to know its **machine_guid** or have a **person_guid** it is linked to it.

## Who talks to the registry?

Your web browser **only**! Check here if this is against your policies: [how to not send any information to a thirdparty server](https://github.com/netdata/netdata/wiki/netdata-security#registry-or-how-to-not-send-any-information-to-a-thirdparty-server)

Your netdata servers do not talk to the registry. This is a UML diagram of its operation:

![registry](https://cloud.githubusercontent.com/assets/2662304/19448565/11a70632-94ab-11e6-9d80-f410b4acb797.png)

## What data the registry maintains?

Its database contains:

- **random person GUIDs** (generated by the registry as a browser cookie)
- **random machine GUIDs** (generated by each netdata server on its first run), including the hostname of the server netdata is running (without the domain)
- **URLs** (the base URL for accessing a netdata server, as seen by the web browser)

For *persons* and *machines*, the registry keeps links to *URLs*, each link with 2 timestamps (first time seen, last time seen) and a counter (number of times it has been seen).

## Which is the default registry?

`https://registry.my-netdata.io`, which is currently served by `https://london.my-netdata.io`. This registry listens to both HTTP and HTTPS requests but the default is HTTPS.

### Can this registry handle the global load of netdata installations?

Yeap! The registry can handle 50.000 - 100.000 requests **per second per core** (depending on the type of CPU, the computer's memory bandwidth, etc). 50.000 is on J1900 (celeron 2Ghz).

We believe, it can do it...

## Every netdata can be a registry

Yes, you read correct, **every netdata can be a registry**. Just pick one and configure it.

**To turn any netdata into a registry**, edit `/etc/netdata/netdata.conf` and set:

```
[registry]
    enabled = yes
    registry to announce = http://your.registry:19999
```

Restart your netdata to activate it.

Then, you need to tell **all your other netdata servers to advertise your registry**, instead of the default. To do this, on each of your netdata servers, edit `/etc/netdata/netdata.conf` and set:

```
[registry]
    enabled = no
    registry to announce = http://your.registry:19999
```

Note that we have not enabled the registry on the other servers. Only one netdata (the registry) needs `[registry].enabled = yes`.

This is it. You have your registry now.

You may also want to give your server different names under the **my-netdata** menu (i.e. to have them sorted / grouped). You can change its registry name, by setting on each netdata server:

```
[registry]
    registry hostname = Group1 - Master DB
```

So this server will appear in **my-netdata** as `Group1 - Master DB`. The max name length is 50 characters.

### Limiting access to the registry

netdata v1.9+ support limiting access to the registry from given IPs, like this:
```
[registry]
    allow from = *
```

`allow from` settings are [netdata simple patterns](https://github.com/netdata/netdata/wiki/Configuration#netdata-simple-patterns): string matches that use `*` as wildcard (any number of times) and a `!` prefix for a negative match. So: `allow from = !10.1.2.3 10.*` will allow all IPs in `10.*` except `10.1.2.3`. The order is important: left to right, the first positive or negative match is used.

Keep in mind that connections to netdata API ports are filtered by `[web].allow connections from`. So, IPs allowed by `[registry].allow from` should also be allowed by `[web].allow connection from`.

### Where is the registry database stored?

`/var/lib/netdata/registry/*.db`

There can be up to 2 files:

- `registry-log.db`, the transaction log

    all incoming requests that affect the registry are saved in this file in real-time.

- `registry.db`, the database

    every `[registry].registry save db every new entries` entries in `registry-log.db`, netdata will save its database to `registry.db` and empty `registry-log.db`.

Both files are machine readable text files.

## The future

The registry opens a whole world of new possibilities for netdata. Check here what we think: https://github.com/netdata/netdata/issues/416

## Troubleshooting the registry

The registry URL should be set to the URL of a netdata dashboard. This server has to have `[registry].enabled = yes`. So, accessing the registry URL directly with your web browser, should present the dashboard of the netdata operating the registry.

To use the registry, your web browser needs to support **third party cookies**, since the cookies are set by the registry while you are browsing the dashboard of another netdata server. The registry, the first time it sees a new web browser it tries to figure if the web browser has cookies enabled or not. It does this by setting a cookie and redirecting the browser back to itself hoping that it will receive the cookie. If it does not receive the cookie, the registry will keep redirecting your web browser back to itself, which after a few redirects will fail with an error like this:

```
ERROR 409: Cannot ACCESS netdata registry: https://registry.my-netdata.io responded with: {"status":"redirect","registry":"https://registry.my-netdata.io"}
```

This error is printed on your web browser console (press F12 on your browser to see it).
