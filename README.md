# NOTE

A blog post with more updated info then this has been posted [here](https://daniel.haxx.se/blog/2018/06/03/inside-firefoxs-doh-engine/)

# Preferences

All preferences for the DNS-over-HTTPS functionality in Firefox are located under the "network.trr" prefix (TRR == Trusted Recursive Resolver). The support for these are targeted for shipping in release Firefox 62.

## network.trr.mode 
set which resolver mode you want.

0 - **Off** (default). use standard native resolving only (don't use TRR at all)

1 - **Race** native against TRR. Do them both in parallel and go with the one that returns a result first.

2 - **First**. Use TRR first, and only if the name resolve fails use the native resolver as a fallback.

3 - **Only**. Only use TRR. Never use the native (after the initial setup).

4 - **Shadow**. Runs the TRR resolves in parallel with the native for timing and measurements but uses only the native resolver results.

5 - **Off by choice** This is the same as 0 but marks it as done by choice and not done by default.

## network.trr.uri

(default: none) set the URI for your DOH server. That's the URL Firefox will issue its HTTP request to. It must be a HTTPS URL. If "useGET" is enabled, Firefox will append "?ct&dns=...." to the URI when it makes its HTTP requests. For the default POST requests, they will be issued to exactly the specified URI.

Publicly announced servers include:
- https://mozilla.cloudflare-dns.com/dns-query
- https://dns.google.com/experimental

## network.trr.credentials

(default: none) set credentials that will be used in the HTTP requests to the DOH end-point. It is the right side content, the value, sent in the Authorization: request header.

## network.trr.wait-for-portal

(default: true) set this boolean to tell Firefox true to wait for the captive portal detection to okay first before TRR is used. (on Android, this will default to **false** since the captive portal handling is done outside of Firefox, by the OS itself.)

## network.trr.allow-rfc1918

(default: false) set this to true to allow RFC 1918 private addresses in TRR responses. When set false, any such response will be considered a wrong response that won't be used.

## network.trr.useGET

(default: false) When the browser issues a request to the DOH server to resolve host names, it can do that using POST or GET. By default Firefox will use POST, but by toggling this you can enforce GET to be used instead.

## network.trr.confirmationNS

(default: example.com) Firefox will check an NS entry at startup to verify that TRR works to ensure proper configuration. This preference sets which domain to check. The verification only checks for a positive answer, it doesn't actually care what the response data says. Set this to `skip` to completely avoid confirmation.

## network.trr.bootstrapAddress

(default: none) by setting this field to the IP address of the host name used in "network.trr.uri", you can bypass using the system native resolver for it.

## network.trr.blacklist-duration

(default: 60) is the number of seconds a name will be kept in the TRR blacklist until it expires and then will be tried with TRR again. The default duration is one minute.

Entries are added to the TRR blacklist when the resolve fails with TRR but works with the native resolver, or if the subsequent connection with a TRR resolved host name fails but works with a retry that is resolved natively. When a host name is added to the TRR, its domain gets checked in the background to see if the whole domain should be blacklisted to ensure a smoother ride going forward.

## network.trr.request-timeout

(default: 3000) is the number of milliseconds a request to and corresponding response from the DOH server is allowed to take until considered failed and discarded.

## network.trr.early-AAAA

(default: false) For each normal name resolve, Firefox issues one HTTP request for A entries and another for AAAA entries. The responses come back separately and can come in any order. If the A records arrive first, Firefox will - as an optimization - continue and use those without waiting for the second response. If the AAAA records arrive first, Firefox will only continue and use them immediately if this option is set to **true**.

# Does it work?

Go to about:networking, click the DNS link in the left-side menu. That shows the contents of the in-memory DNS cache. The TRR column says "true" for host names that were resolved using TRR (DNS-over-HTTPS).

# I found a bug!

If you experience a problem or find a host name that has problems with TRR, consider extracting logs from Firefox during the hiccup by asking for `nsHostResolver:5` logs as instructed on the [HTTP logging page](https://developer.mozilla.org/en-US/docs/Mozilla/Debugging/HTTP_logging) and [submit a bug report](https://bugzilla.mozilla.org/enter_bug.cgi?assigned_to=nobody%40mozilla.org&bug_file_loc=http%3A%2F%2F&bug_ignored=0&bug_severity=normal&bug_status=NEW&cf_blocking_fennec=---&cf_fx_iteration=---&cf_fx_points=---&cf_platform_rel=---&cf_status_firefox59=---&cf_status_firefox60=---&cf_status_firefox61=affected&cf_status_firefox_esr52=---&cf_status_thunderbird_esr52=---&cf_tracking_firefox60=---&cf_tracking_firefox61=---&cf_tracking_firefox_esr52=---&cf_tracking_firefox_relnote=---&cf_tracking_thunderbird_esr52=---&component=Networking%3A%20DNS&contenttypemethod=autodetect&contenttypeselection=text%2Fplain&defined_groups=1&flag_type-203=X&flag_type-37=X&flag_type-4=X&flag_type-41=X&flag_type-5=X&flag_type-607=X&flag_type-721=X&flag_type-737=X&flag_type-787=X&flag_type-799=X&flag_type-800=X&flag_type-803=X&flag_type-835=X&flag_type-846=X&flag_type-855=X&flag_type-863=X&flag_type-864=X&flag_type-914=X&flag_type-916=X&form_name=enter_bug&maketemplate=Remember%20values%20as%20bookmarkable%20template&op_sys=Unspecified&priority=--&product=Core&rep_platform=Unspecified&target_milestone=---&version=Trunk)
