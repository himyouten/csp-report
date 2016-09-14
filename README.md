# CSP Report consumer

The CSP Report consumer is an http endpoint that the report-uri can use.  It simply writes the json payload to a log file to be processed by any log analysis system, e.g. Scalyr.  We had used a 3rd party report service but we overloaded their thresholds so were not getting reports.  The added advantage of setting up our own CSP report consumer was that it keeps everything internal, albeit using an log aggregator service.

The CSP report consumer is just an Nginx server block config.  [Nginx](http://nginx.org/en/) is an extremely versatile HTTP server.

Also included are the Scalyr agent and parser configs.  [Scalyr](https://www.scalyr.com/product) is a great log aggregator, easy to setup and use.

## Nginx setup
First, you need to define the new log format.  As the CSP json is POSTed, we need to capture the request body.
```
# csp-report log format
log_format csp-report '[$time_local] $remote_addr "$http_referer" $request_body';
```

The main server block simply uses the new log format.  Only thing tricky here, is that Nginx will not populate `$request_body` unless it passes through either `proxy_pass` or `fastcgi_pass`.  I got the basis to this solution at [StackOverflow](http://stackoverflow.com/questions/4939382/logging-post-data-from-request-body), as one does!  I chose to use `proxy_pass` as I didn't want to install php-fpm or any thing else on the server.

```
# main endpoint
server {
  server_name csp-report.mydomain.com;

  location / {
    access_log /var/log/nginx/access.log main;
    access_log /var/log/nginx/csp-report.log csp-report;
    # required for nginx to read request body, otherwise, its empty
    proxy_pass http://localhost:8080;
  }
}

# required for nginx to read request body, otherwise, its empty
server {
  server_name csp-report.mydomain.com;
  listen 8080;

  location / {
    return 200 "Logged";
  }
}
```

## Processing with Scalyr
I've been using Scalyr for a some time now and its proven itself to be quite useful at analysing logs.  It even has the ability to graph the rates make it extremely useful for checking trends.  But feel free to use whatever tool/system you want to digest the logs.

Scalyr has two parts, the agent config and server parser.

For the agent config, the only thing extra added was the replacement of the `\x22` encoded character which Nginx happens to put in the `$request_body`.  I simply replace back with the original double-quote.  It's set up in the `redaction_rules` but is not really a redaction.

```
redaction_rules: [
  // replace \x22 with '"'
  {
    match_expression: "\\\\x22",
    replacement: "\""
},
],
```

And finally, the server parser is dead simple.  Scalyr has some very nice built in field parsers, one being `json`.  And as we fixed up the `\x22`, and the log is now a proper json again, the `json` field parser can be used as is.

```
format: "\\[$timestamp$\\] $remoteIp$ $referrer=quotable$ $report{parse=json}$",
```

## Complete configs

*  [nginx-csp-report.conf](../blob/master/nginx-csp-report.conf)
*  [scalyr-agent-conf.json](../blob/master/scalyr-agent-conf.json)
*  [scalyr-agent-conf.json](../blob/master/scalyr-agent-conf.json)
