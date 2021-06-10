# Unofficial docs

This is unofficial docs. The official TRR docs are in [Mozilla's
wiki](https://wiki.mozilla.org/Trusted_Recursive_Resolver) - and is largely
based on the information we've collected here.

# Preferences

All preferences for the DNS-over-HTTPS functionality in Firefox are located under the `network.trr` prefix (TRR == Trusted Recursive Resolver). The support for these were added in Firefox 62.

## network.trr.mode 
set which resolver mode you want.

0 - **Off** (default). use standard native resolving only (don't use TRR at all)

1 - **Race** (removed)

2 - **First**. Use TRR first, and only if the name resolve fails use the native resolver as a fallback.

3 - **Only**. Only use TRR. Never use the native (after the initial setup).

  - **Note:** if using a hostname, `network.trr.bootstrapAddress` must be set for resolution to work in TRR mode 3. See https://bugzilla.mozilla.org/show_bug.cgi?id=1600976

4 - **Shadow**. (removed)

5 - **Off by choice** This is the same as 0 but marks it as done by choice and not done by default.

## network.trr.uri

(default: none) set the URI for your DoH server. That's the URL Firefox will issue its HTTP request to. It must be a HTTPS URL. If "useGET" is enabled, Firefox will append "?dns=...." to the URI when it makes its HTTP requests. For the default POST requests, they will be issued to exactly the specified URI.

Publicly announced servers include:
- https://mozilla.cloudflare-dns.com/dns-query
- https://dns.google/dns-query

For more servers, see the unofficial [list of DoH servers](https://github.com/curl/curl/wiki/DNS-over-HTTPS).

## network.trr.credentials

(default: none) set credentials that will be used in the HTTP requests to the DoH end-point. It is the right-hand side content, the value, sent in the Authorization: request header.

## network.trr.wait-for-portal

(default: true) set this boolean to **true** to tell Firefox to wait for the captive portal detection before TRR is used. (on Android, this will default to **false** since the captive portal handling is done outside of Firefox, by the OS itself.)

## network.trr.allow-rfc1918

(default: false) set this to true to allow RFC 1918 private addresses in TRR responses. When set to false, any such response will be considered invalid and won't be used.

## network.trr.useGET

(default: false) When the browser issues a request to the DoH server to resolve host names, it can do that using POST or GET. By default Firefox will use POST, but by toggling this you can enforce GET to be used instead.

## network.trr.confirmationNS

(default: example.com) Firefox will check an NS entry at startup to verify that TRR works to ensure proper configuration. This preference sets which domain to check. The verification only checks for a positive answer, it doesn't actually care what the response data says. Set this to `skip` to completely avoid confirmation.

## network.trr.bootstrapAddr

(default: none) by setting this field to the IP address of the host name used in "network.trr.uri", you can bypass using the system native resolver for it.

**NOTE:** Before Firefox 89, this was named `network.trr.bootstrapAddress`.

## network.trr.blacklist-duration

(default: 60) is the number of seconds a name will be kept in the TRR blacklist until it expires and then will be tried with TRR again. The default duration is one minute.

Entries are added to the TRR blacklist when the resolution fails with TRR but works with the native resolver, or if the subsequent connection with a TRR resolved host name fails but works with a retry that is resolved natively. When a hostname is added to the TRR, its domain gets checked in the background to see if the whole domain should be blacklisted to ensure a smoother ride going forward.

## network.trr.request-timeout

(default: 3000) is the number of milliseconds a request and the corresponding response from the DoH server is allowed to take until considered failed and discarded.

## network.trr.early-AAAA

(default: false) For each normal name resolution, Firefox issues one HTTP request for A entries and another for AAAA entries. The responses come back separately and can come in any order. If the A records arrive first, Firefox will—as an optimization— continue and use them without waiting for the second response. If the AAAA records arrive first, Firefox will only continue and use them immediately if this option is set to **true**.

## network.trr.max-fails

(default: 5) If this many DoH requests fail in a row, consider TRR broken and go back to verify-NS state. This is meant to detect situations when the DoH server dies.

## network.trr.disable-ECS

(default: true) If set, TRR asks the resolver to disable ECS (EDNS Client Subnet: the method where the resolver passes on the subnet of the client asking the question). Some resolvers will use ECS to the upstream if this request is not passed on to them.

## network.trr.excluded-domains

(default: `localhost,local`) Comma separated list of domain names to be resolved using the native resolver instead of TRR.

## network.trr.resolvers

(default: `[{ "name": "Cloudflare", "url": "https://mozilla.cloudflare-dns.com/dns-query" }]`) JSON list of DoH resolver providers that should be presented in the preferences GUI. Each item is an object with "name" and "url" keys. Modifying this should not be necessary unless you want to update the preferences GUI.

## network.trr.custom_uri

(default: none) The custom DoH URI supplied by the user in the GUI preferences ("Custom"). This is not necessarily the same URI as will be used for DoH resolution, but is only used to store the user supplied text field value. See `network.trr.uri` for the setting that actually affects resolution. Modifying this should not be necessary unless you want to update the preferences GUI.

# Split-horizon and blacklist

With regular DNS, it is common to have clients in different places get different results back. This can be done since the servers know where the request are coming frome and they can then respond accordingly. When switching to another resolver with TRR, you may experience that you don’t always get the same set of addresses back. At times, this causes problems.

As a precaution, Firefox features a system that detects if a name can’t be resolved at all with TRR and can retry with the native resolver (the so called TRR-first mode). Ending up in this scenario is slower and leaks the name in cleartext, but this safety mechanism exists to avoid users risking ending up in a black hole where certain sites can’t be accessed. Names that cause such TRR failures are then put into a dynamic blacklist so that subsequent uses of that name automatically avoid using DoH for a while (see the `blacklist-duration` pref to control that period). This fallback is not used if TRR-only mode is selected.

In addition, if a host's address is retrieved via TRR and Firefox subsequently fails to connect to that host, it will redo the resolution without DoH and retry the connection to ensure t it wasn't a split horizon setup that caused the problem.

When a hostname is added to the TRR blacklist, its domain also gets checked in the background to see if that whole domain perhaps should be blacklisted to ensure a smoother ride going forward.

Additionally, “localhost” and all names in the “.local” TLD are sort of hard-coded as blacklisted and will never be resolved with TRR. (Unless you run TRR-only…)

# Bootstrap

You specify the DoH service as a full URI with a name that needs to be resolved. On cold start Firefox won’t know the IP address to that name so it needs to resolve it first (unless `network.trr.bootstrapAddress` is set). Firefox will use the native resolver until TRR has proven itself to work by resolving the test domain configured by `network.trr.confirmationNS` preference. Firefox will also by default wait for the captive portal check to signal “OK” before it uses TRR, unless you tell it otherwise by `network.trr.wait-for-portal` preference.

As a result of this bootstrap procedure, and if you’re not in TRR-only mode, you might still get  a few native name resolves done at initial Firefox startups. Just telling you this so you don’t panic if you see a few show up.

# about:networking

Go to about:networking, click the DNS link in the left-side menu. That shows the contents of the in-memory DNS cache. The TRR column says "true" for host names that were resolved using TRR (DNS-over-HTTPS).

# I found a bug!

If you experience a problem or find a host name that has problems with TRR, consider extracting logs from Firefox during the hiccup by asking for `nsHostResolver:5` logs as instructed on the [HTTP logging page](https://developer.mozilla.org/en-US/docs/Mozilla/Debugging/HTTP_logging) and [submit a bug report](https://bugzilla.mozilla.org/enter_bug.cgi?assigned_to=nobody%40mozilla.org&bug_file_loc=http%3A%2F%2F&bug_ignored=0&bug_severity=normal&bug_status=NEW&cf_blocking_fennec=---&cf_fx_iteration=---&cf_fx_points=---&cf_platform_rel=---&cf_status_firefox59=---&cf_status_firefox60=---&cf_status_firefox61=affected&cf_status_firefox_esr52=---&cf_status_thunderbird_esr52=---&cf_tracking_firefox60=---&cf_tracking_firefox61=---&cf_tracking_firefox_esr52=---&cf_tracking_firefox_relnote=---&cf_tracking_thunderbird_esr52=---&component=Networking%3A%20DNS&contenttypemethod=autodetect&contenttypeselection=text%2Fplain&defined_groups=1&flag_type-203=X&flag_type-37=X&flag_type-4=X&flag_type-41=X&flag_type-5=X&flag_type-607=X&flag_type-721=X&flag_type-737=X&flag_type-787=X&flag_type-799=X&flag_type-800=X&flag_type-803=X&flag_type-835=X&flag_type-846=X&flag_type-855=X&flag_type-863=X&flag_type-864=X&flag_type-914=X&flag_type-916=X&form_name=enter_bug&maketemplate=Remember%20values%20as%20bookmarkable%20template&op_sys=Unspecified&priority=--&product=Core&rep_platform=Unspecified&target_milestone=---&version=Trunk)
