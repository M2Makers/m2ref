
.. program:: M2

Global Componenets
==========

SYNOPSIS
--------

**nghttpx** [OPTIONS]... [<PRIVATE_KEY> <CERT>]

DESCRIPTION
-----------

A reverse proxy for HTTP/2, and HTTP/1.

.. describe:: <PRIVATE_KEY>

    
    Set  path  to  server's private  key.   Required  unless
    "no-tls" parameter is used in :option:`--frontend` option.

.. describe:: <CERT>

    Set  path  to  server's  certificate.   Required  unless
    "no-tls"  parameter is  used in  :option:`--frontend` option.   To
    make OCSP stapling work, this must be an absolute path.


OPTIONS
-------

The options are categorized into several groups.

Connections
~~~~~~~~~~~

.. option:: -b, --backend=(<HOST>,<PORT>|unix:<PATH>)[;[<PATTERN>[:...]][[;<PARAM>]...]


    Set  backend  host  and   port.   The  multiple  backend
    addresses are  accepted by repeating this  option.  UNIX
    domain socket  can be  specified by prefixing  path name
    with "unix:" (e.g., unix:/var/run/backend.sock).

    Optionally, if <PATTERN>s are given, the backend address
    is  only  used  if  request matches  the  pattern.   The
    pattern  matching is  closely  designed  to ServeMux  in
    net/http package of  Go programming language.  <PATTERN>
    consists of  path, host +  path or just host.   The path
    must start  with "*/*".  If  it ends with "*/*",  it matches
    all  request path  in  its subtree.   To  deal with  the
    request  to the  directory without  trailing slash,  the
    path which ends  with "*/*" also matches  the request path
    which  only  lacks  trailing  '*/*'  (e.g.,  path  "*/foo/*"
    matches request path  "*/foo*").  If it does  not end with
    "*/*", it  performs exact match against  the request path.
    If  host  is given,  it  performs  a match  against  the
    request host.   For a  request received on  the frontend
    listener with  "sni-fwd" parameter enabled, SNI  host is
    used instead of a request host.  If host alone is given,
    "*/*" is  appended to it,  so that it matches  all request
    paths  under the  host  (e.g., specifying  "nghttp2.org"
    equals  to "nghttp2.org/").   CONNECT method  is treated
    specially.  It  does not have  path, and we  don't allow
    empty path.  To workaround  this, we assume that CONNECT
    method has "*/*" as path.

    Patterns with  host take  precedence over  patterns with
    just path.   Then, longer patterns take  precedence over
    shorter ones.

    Host  can  include "\*"  in  the  left most  position  to
    indicate  wildcard match  (only suffix  match is  done).
    The "\*" must match at least one character.  For example,
    host    pattern    "\*.nghttp2.org"    matches    against
    "www.nghttp2.org"  and  "git.ngttp2.org", but  does  not
    match  against  "nghttp2.org".   The exact  hosts  match
    takes precedence over the wildcard hosts match.

    If path  part ends with  "\*", it is treated  as wildcard
    path.  The  wildcard path  behaves differently  from the
    normal path.  For normal path,  match is made around the
    boundary of path component  separator,"*/*".  On the other
    hand, the wildcard  path does not take  into account the
    path component  separator.  All paths which  include the
    wildcard  path  without  last  "\*" as  prefix,  and  are
    strictly longer than wildcard  path without last "\*" are
    matched.  "\*"  must match  at least one  character.  For
    example,  the   pattern  "*/foo\**"  matches   "*/foo/*"  and
    "*/foobar*".  But it does not match "*/foo*", or "*/fo*".

    If <PATTERN> is omitted or  empty string, "*/*" is used as
    pattern,  which  matches  all request  paths  (catch-all
    pattern).  The catch-all backend must be given.

    When doing  a match, nghttpx made  some normalization to
    pattern, request host and path.  For host part, they are
    converted to lower case.  For path part, percent-encoded
    unreserved characters  defined in RFC 3986  are decoded,
    and any  dot-segments (".."  and ".")   are resolved and
    removed.

    For   example,   :option:`-b`\'127.0.0.1,8080;nghttp2.org/httpbin/'
    matches the  request host "nghttp2.org" and  the request
    path "*/httpbin/get*", but does not match the request host
    "nghttp2.org" and the request path "*/index.html*".

    The  multiple <PATTERN>s  can  be specified,  delimiting
    them            by           ":".             Specifying
    :option:`-b`\'127.0.0.1,8080;nghttp2.org:www.nghttp2.org'  has  the
    same  effect  to specify  :option:`-b`\'127.0.0.1,8080;nghttp2.org'
    and :option:`-b`\'127.0.0.1,8080;www.nghttp2.org'.

    The backend addresses sharing same <PATTERN> are grouped
    together forming  load balancing  group.

    Several parameters <PARAM> are accepted after <PATTERN>.
    The  parameters are  delimited  by  ";".  The  available
    parameters       are:      "proto=<PROTO>",       "tls",
    "sni=<SNI_HOST>",         "fall=<N>",        "rise=<N>",
    "affinity=<METHOD>",    "dns",    "redirect-if-not-tls",
    "upgrade-scheme",                        "mruby=<PATH>",
    "read-timeout=<DURATION>",   "write-timeout=<DURATION>",
    "group=<GROUP>",  "group-weight=<N>", and  "weight=<N>".
    The  parameter  consists   of  keyword,  and  optionally
    followed by  "=" and value.  For  example, the parameter
    "proto=h2"  consists of  the keyword  "proto" and  value
    "h2".  The parameter "tls" consists of the keyword "tls"
    without value.  Each parameter is described as follows.

    The backend application protocol  can be specified using
    optional  "proto"   parameter,  and   in  the   form  of
    "proto=<PROTO>".  <PROTO> should be one of the following
    list  without  quotes:  "h2", "http/1.1".   The  default
    value of <PROTO> is  "http/1.1".  Note that usually "h2"
    refers to HTTP/2  over TLS.  But in this  option, it may
    mean HTTP/2  over cleartext TCP unless  "tls" keyword is
    used (see below).

    TLS  can   be  enabled  by  specifying   optional  "tls"
    parameter.  TLS is not enabled by default.

    With "sni=<SNI_HOST>" parameter, it can override the TLS
    SNI  field  value  with  given  <SNI_HOST>.   This  will
    default to the backend <HOST> name

    The  feature  to detect  whether  backend  is online  or
    offline can be enabled  using optional "fall" and "rise"
    parameters.   Using  "fall=<N>"  parameter,  if  nghttpx
    cannot connect  to a  this backend <N>  times in  a row,
    this  backend  is  assumed  to be  offline,  and  it  is
    excluded from load balancing.  If <N> is 0, this backend
    never  be excluded  from load  balancing whatever  times
    nghttpx cannot connect  to it, and this  is the default.
    There is  also "rise=<N>" parameter.  After  backend was
    excluded from load balancing group, nghttpx periodically
    attempts to make a connection to the failed backend, and
    if the  connection is made  successfully <N> times  in a
    row, the backend is assumed to  be online, and it is now
    eligible  for load  balancing target.   If <N>  is 0,  a
    backend  is permanently  offline, once  it goes  in that
    state, and this is the default behaviour.

    The     session     affinity    is     enabled     using
    "affinity=<METHOD>"  parameter.   If  "ip" is  given  in
    <METHOD>, client  IP based session affinity  is enabled.
    If "cookie"  is given in <METHOD>,  cookie based session
    affinity is  enabled.  If  "none" is given  in <METHOD>,
    session affinity  is disabled, and this  is the default.
    The session  affinity is  enabled per <PATTERN>.   If at
    least  one backend  has  "affinity"  parameter, and  its
    <METHOD> is not "none",  session affinity is enabled for
    all backend  servers sharing the same  <PATTERN>.  It is
    advised  to  set  "affinity" parameter  to  all  backend
    explicitly if session affinity  is desired.  The session
    affinity  may   break  if   one  of  the   backend  gets
    unreachable,  or   backend  settings  are   reloaded  or
    replaced by API.

    If   "affinity=cookie"    is   used,    the   additional
    configuration                is                required.
    "affinity-cookie-name=<NAME>" must be  used to specify a
    name     of     cookie      to     use.      Optionally,
    "affinity-cookie-path=<PATH>" can  be used to  specify a
    path   which   cookie    is   applied.    The   optional
    "affinity-cookie-secure=<SECURE>"  controls  the  Secure
    attribute of a cookie.  The default value is "auto", and
    the Secure attribute is  determined by a request scheme.
    If a request scheme is "https", then Secure attribute is
    set.  Otherwise, it  is not set.  If  <SECURE> is "yes",
    the  Secure attribute  is  always set.   If <SECURE>  is
    "no", the Secure attribute is always omitted.

    By default, name resolution of backend host name is done
    at  start  up,  or reloading  configuration.   If  "dns"
    parameter   is  given,   name  resolution   takes  place
    dynamically.  This is useful  if backend address changes
    frequently.   If  "dns"  is given,  name  resolution  of
    backend   host   name   at  start   up,   or   reloading
    configuration is skipped.

    If "redirect-if-not-tls" parameter  is used, the matched
    backend  requires   that  frontend  connection   is  TLS
    encrypted.  If it isn't, nghttpx responds to the request
    with 308  status code, and  https URI the  client should
    use instead  is included in Location  header field.  The
    port number in  redirect URI is 443 by  default, and can
    be  changed using  :option:`--redirect-https-port` option.   If at
    least one  backend has  "redirect-if-not-tls" parameter,
    this feature is enabled  for all backend servers sharing
    the   same   <PATTERN>.    It    is   advised   to   set
    "redirect-if-no-tls"    parameter   to    all   backends
    explicitly if this feature is desired.

    If "upgrade-scheme"  parameter is used along  with "tls"
    parameter, HTTP/2 :scheme pseudo header field is changed
    to "https" from "http" when forwarding a request to this
    particular backend.  This is  a workaround for a backend
    server  which  requires  "https" :scheme  pseudo  header
    field on TLS encrypted connection.

    "mruby=<PATH>"  parameter  specifies  a  path  to  mruby
    script  file  which  is  invoked when  this  pattern  is
    matched.  All backends which share the same pattern must
    have the same mruby path.

    "read-timeout=<DURATION>" and "write-timeout=<DURATION>"
    parameters  specify the  read and  write timeout  of the
    backend connection  when this  pattern is  matched.  All
    backends which share the same pattern must have the same
    timeouts.  If these timeouts  are entirely omitted for a
    pattern,            :option:`--backend-read-timeout`           and
    :option:`--backend-write-timeout` are used.

    "group=<GROUP>"  parameter specifies  the name  of group
    this backend address belongs to.  By default, it belongs
    to  the unnamed  default group.   The name  of group  is
    unique   per   pattern.   "group-weight=<N>"   parameter
    specifies the  weight of  the group.  The  higher weight
    gets  more frequently  selected  by  the load  balancing
    algorithm.  <N> must be  [1, 256] inclusive.  The weight
    8 has 4 times more weight  than 2.  <N> must be the same
    for  all addresses  which  share the  same <GROUP>.   If
    "group-weight" is  omitted in an address,  but the other
    address  which  belongs  to  the  same  group  specifies
    "group-weight",   its    weight   is   used.     If   no
    "group-weight"  is  specified  for  all  addresses,  the
    weight of a group becomes 1.  "group" and "group-weight"
    are ignored if session affinity is enabled.

    "weight=<N>"  parameter  specifies  the  weight  of  the
    backend  address  inside  a  group  which  this  address
    belongs  to.  The  higher  weight  gets more  frequently
    selected by  the load balancing algorithm.   <N> must be
    [1,  256] inclusive.   The  weight 8  has  4 times  more
    weight  than weight  2.  If  this parameter  is omitted,
    weight  becomes  1.   "weight"  is  ignored  if  session
    affinity is enabled.

    Since ";" and ":" are  used as delimiter, <PATTERN> must
    not  contain these  characters.  Since  ";" has  special
    meaning in shell, the option value must be quoted.


    Default: ``127.0.0.1,80``

.. option:: -f, --frontend=(<HOST>,<PORT>|unix:<PATH>)[[;<PARAM>]...]

    Set  frontend  host and  port.   If  <HOST> is  '\*',  it
    assumes  all addresses  including  both  IPv4 and  IPv6.
    UNIX domain  socket can  be specified by  prefixing path
    name  with  "unix:" (e.g.,  unix:/var/run/nghttpx.sock).
    This  option can  be used  multiple times  to listen  to
    multiple addresses.

    This option  can take  0 or  more parameters,  which are
    described  below.   Note   that  "api"  and  "healthmon"
    parameters are mutually exclusive.

    Optionally, TLS  can be disabled by  specifying "no-tls"
    parameter.  TLS is enabled by default.

    If "sni-fwd" parameter is  used, when performing a match
    to select a backend server,  SNI host name received from
    the client  is used  instead of  the request  host.  See
    :option:`--backend` option about the pattern match.

    To  make this  frontend as  API endpoint,  specify "api"
    parameter.   This   is  disabled  by  default.    It  is
    important  to  limit the  access  to  the API  frontend.
    Otherwise, someone  may change  the backend  server, and
    break your services,  or expose confidential information
    to the outside the world.

    To  make  this  frontend  as  health  monitor  endpoint,
    specify  "healthmon"  parameter.   This is  disabled  by
    default.  Any  requests which come through  this address
    are replied with 200 HTTP status, without no body.

    To accept  PROXY protocol  version 1  and 2  on frontend
    connection,  specify  "proxyproto" parameter.   This  is
    disabled by default.


    Default: ``*,3000``

.. option:: --backlog=<N>

    Set listen backlog size.

    Default: ``65536``

.. option:: --backend-address-family=(auto|IPv4|IPv6)

    Specify  address  family  of  backend  connections.   If
    "auto" is given, both IPv4  and IPv6 are considered.  If
    "IPv4" is  given, only  IPv4 address is  considered.  If
    "IPv6" is given, only IPv6 address is considered.

    Default: ``auto``

.. option:: --backend-http-proxy-uri=<URI>

    Specify      proxy       URI      in       the      form
    http://[<USER>:<PASS>@]<PROXY>:<PORT>.    If   a   proxy
    requires  authentication,  specify  <USER>  and  <PASS>.
    Note that  they must be properly  percent-encoded.  This
    proxy  is used  when the  backend connection  is HTTP/2.
    First,  make  a CONNECT  request  to  the proxy  and  it
    connects  to the  backend  on behalf  of nghttpx.   This
    forms  tunnel.   After  that, nghttpx  performs  SSL/TLS
    handshake with  the downstream through the  tunnel.  The
    timeouts when connecting and  making CONNECT request can
    be     specified    by     :option:`--backend-read-timeout`    and
    :option:`--backend-write-timeout` options.


Performance
~~~~~~~~~~~

.. option:: -n, --workers=<N>

    Set the number of worker threads.

    Default: ``1``

.. option:: --single-thread

    Run everything in one  thread inside the worker process.
    This   feature   is   provided  for   better   debugging
    experience,  or  for  the platforms  which  lack  thread
    support.   If  threading  is disabled,  this  option  is
    always enabled.

.. option:: --read-rate=<SIZE>

    Set maximum  average read  rate on  frontend connection.
    Setting 0 to this option means read rate is unlimited.

    Default: ``0``

.. option:: --read-burst=<SIZE>

    Set  maximum read  burst  size  on frontend  connection.
    Setting  0  to this  option  means  read burst  size  is
    unlimited.

    Default: ``0``

.. option:: --write-rate=<SIZE>

    Set maximum  average write rate on  frontend connection.
    Setting 0 to this option means write rate is unlimited.

    Default: ``0``

.. option:: --write-burst=<SIZE>

    Set  maximum write  burst size  on frontend  connection.
    Setting  0 to  this  option means  write  burst size  is
    unlimited.

    Default: ``0``

.. option:: --worker-read-rate=<SIZE>

    Set maximum average read rate on frontend connection per
    worker.  Setting  0 to  this option  means read  rate is
    unlimited.  Not implemented yet.

    Default: ``0``

.. option:: --worker-read-burst=<SIZE>

    Set maximum  read burst size on  frontend connection per
    worker.  Setting 0 to this  option means read burst size
    is unlimited.  Not implemented yet.

    Default: ``0``

.. option:: --worker-write-rate=<SIZE>

    Set maximum  average write  rate on  frontend connection
    per worker.  Setting  0 to this option  means write rate
    is unlimited.  Not implemented yet.

    Default: ``0``

.. option:: --worker-write-burst=<SIZE>

    Set maximum write burst  size on frontend connection per
    worker.  Setting 0 to this option means write burst size
    is unlimited.  Not implemented yet.

    Default: ``0``

.. option:: --worker-frontend-connections=<N>

    Set maximum number  of simultaneous connections frontend
    accepts.  Setting 0 means unlimited.

    Default: ``0``

.. option:: --backend-connections-per-host=<N>

    Set  maximum number  of  backend concurrent  connections
    (and/or  streams in  case  of HTTP/2)  per origin  host.
    This option  is meaningful when :option:`--http2-proxy`  option is
    used.   The  origin  host  is  determined  by  authority
    portion of  request URI (or :authority  header field for
    HTTP/2).   To  limit  the   number  of  connections  per
    frontend        for       default        mode,       use
    :option:`--backend-connections-per-frontend`\.

    Default: ``8``

.. option:: --backend-connections-per-frontend=<N>

    Set  maximum number  of  backend concurrent  connections
    (and/or streams  in case of HTTP/2)  per frontend.  This
    option  is   only  used  for  default   mode.   0  means
    unlimited.  To limit the  number of connections per host
    with          :option:`--http2-proxy`         option,          use
    :option:`--backend-connections-per-host`\.

    Default: ``0``

.. option:: --rlimit-nofile=<N>

    Set maximum number of open files (RLIMIT_NOFILE) to <N>.
    If 0 is given, nghttpx does not set the limit.

    Default: ``0``

.. option:: --backend-request-buffer=<SIZE>

    Set buffer size used to store backend request.

    Default: ``16K``

.. option:: --backend-response-buffer=<SIZE>

    Set buffer size used to store backend response.

    Default: ``128K``

.. option:: --fastopen=<N>

    Enables  "TCP Fast  Open" for  the listening  socket and
    limits the  maximum length for the  queue of connections
    that have not yet completed the three-way handshake.  If
    value is 0 then fast open is disabled.

    Default: ``0``

.. option:: --no-kqueue

    Don't use  kqueue.  This  option is only  applicable for
    the platforms  which have kqueue.  For  other platforms,
    this option will be simply ignored.


Timeout
~~~~~~~

.. option:: --frontend-http2-read-timeout=<DURATION>

    Specify read timeout for HTTP/2 frontend connection.

    Default: ``3m``

.. option:: --frontend-read-timeout=<DURATION>

    Specify read timeout for HTTP/1.1 frontend connection.

    Default: ``1m``

.. option:: --frontend-write-timeout=<DURATION>

    Specify write timeout for all frontend connections.

    Default: ``30s``

.. option:: --frontend-keep-alive-timeout=<DURATION>

    Specify   keep-alive   timeout   for   frontend   HTTP/1
    connection.

    Default: ``1m``

.. option:: --stream-read-timeout=<DURATION>

    Specify  read timeout  for HTTP/2  streams.  0  means no
    timeout.

    Default: ``0``

.. option:: --stream-write-timeout=<DURATION>

    Specify write  timeout for  HTTP/2 streams.  0  means no
    timeout.

    Default: ``1m``

.. option:: --backend-read-timeout=<DURATION>

    Specify read timeout for backend connection.

    Default: ``1m``

.. option:: --backend-write-timeout=<DURATION>

    Specify write timeout for backend connection.

    Default: ``30s``

.. option:: --backend-connect-timeout=<DURATION>

    Specify  timeout before  establishing TCP  connection to
    backend.

    Default: ``30s``

.. option:: --backend-keep-alive-timeout=<DURATION>

    Specify   keep-alive   timeout    for   backend   HTTP/1
    connection.

    Default: ``2s``

.. option:: --listener-disable-timeout=<DURATION>

    After accepting  connection failed,  connection listener
    is disabled  for a given  amount of time.   Specifying 0
    disables this feature.

    Default: ``30s``

.. option:: --frontend-http2-setting-timeout=<DURATION>

    Specify  timeout before  SETTINGS ACK  is received  from
    client.

    Default: ``10s``

.. option:: --backend-http2-settings-timeout=<DURATION>

    Specify  timeout before  SETTINGS ACK  is received  from
    backend server.

    Default: ``10s``

.. option:: --backend-max-backoff=<DURATION>

    Specify  maximum backoff  interval.  This  is used  when
    doing health  check against offline backend  (see "fail"
    parameter  in :option:`--backend`  option).   It is  also used  to
    limit  the  maximum   interval  to  temporarily  disable
    backend  when nghttpx  failed to  connect to  it.  These
    intervals are calculated  using exponential backoff, and
    consecutive failed attempts increase the interval.  This
    option caps its maximum value.

    Default: ``2m``


SSL/TLS
~~~~~~~

.. option:: --ciphers=<SUITE>

    Set allowed  cipher list  for frontend  connection.  The
    format of the string is described in OpenSSL ciphers(1).
    This option  sets cipher suites for  TLSv1.2 or earlier.
    Use :option:`--tls13-ciphers` for TLSv1.3.

    Default: ``ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256``

.. option:: --tls13-ciphers=<SUITE>

    Set allowed  cipher list  for frontend  connection.  The
    format of the string is described in OpenSSL ciphers(1).
    This  option  sets  cipher   suites  for  TLSv1.3.   Use
    :option:`--ciphers` for TLSv1.2 or earlier.

    Default: ``TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256``

.. option:: --client-ciphers=<SUITE>

    Set  allowed cipher  list for  backend connection.   The
    format of the string is described in OpenSSL ciphers(1).
    This option  sets cipher suites for  TLSv1.2 or earlier.
    Use :option:`--tls13-client-ciphers` for TLSv1.3.

    Default: ``ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256``

.. option:: --tls13-client-ciphers=<SUITE>

    Set  allowed cipher  list for  backend connection.   The
    format of the string is described in OpenSSL ciphers(1).
    This  option  sets  cipher   suites  for  TLSv1.3.   Use
    :option:`--tls13-client-ciphers` for TLSv1.2 or earlier.

    Default: ``TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256``

.. option:: --ecdh-curves=<LIST>

    Set  supported  curve  list  for  frontend  connections.
    <LIST> is a  colon separated list of curve  NID or names
    in the preference order.  The supported curves depend on
    the  linked  OpenSSL  library.  This  function  requires
    OpenSSL >= 1.0.2.

    Default: ``X25519:P-256:P-384:P-521``

.. option:: -k, --insecure

    Don't  verify backend  server's  certificate  if TLS  is
    enabled for backend connections.

.. option:: --cacert=<PATH>

    Set path to trusted CA  certificate file.  It is used in
    backend  TLS connections  to verify  peer's certificate.
    It is also used to  verify OCSP response from the script
    set by :option:`--fetch-ocsp-response-file`\.  The  file must be in
    PEM format.   It can contain multiple  certificates.  If
    the  linked OpenSSL  is configured  to load  system wide
    certificates, they  are loaded at startup  regardless of
    this option.

.. option:: --private-key-passwd-file=<PATH>

    Path  to file  that contains  password for  the server's
    private key.   If none is  given and the private  key is
    password protected it'll be requested interactively.

.. option:: --subcert=<KEYPATH>:<CERTPATH>[[;<PARAM>]...]

    Specify  additional certificate  and  private key  file.
    nghttpx will  choose certificates based on  the hostname
    indicated by client using TLS SNI extension.  If nghttpx
    is  built with  OpenSSL  >= 1.0.2,  the shared  elliptic
    curves (e.g., P-256) between  client and server are also
    taken into  consideration.  This allows nghttpx  to send
    ECDSA certificate  to modern clients, while  sending RSA
    based certificate to older  clients.  This option can be
    used  multiple  times.   To  make  OCSP  stapling  work,
    <CERTPATH> must be absolute path.

    Additional parameter  can be specified in  <PARAM>.  The
    available <PARAM> is "sct-dir=<DIR>".

    "sct-dir=<DIR>"  specifies the  path to  directory which
    contains        \*.sct        files        for        TLS
    signed_certificate_timestamp extension (RFC 6962).  This
    feature   requires   OpenSSL   >=   1.0.2.    See   also
    :option:`--tls-sct-dir` option.

.. option:: --dh-param-file=<PATH>

    Path to file that contains  DH parameters in PEM format.
    Without  this   option,  DHE   cipher  suites   are  not
    available.

.. option:: --npn-list=<LIST>

    Comma delimited list of  ALPN protocol identifier sorted
    in the  order of preference.  That  means most desirable
    protocol comes  first.  This  is used  in both  ALPN and
    NPN.  The parameter must be  delimited by a single comma
    only  and any  white spaces  are  treated as  a part  of
    protocol string.

    Default: ``h2,h2-16,h2-14,http/1.1``

.. option:: --verify-client

    Require and verify client certificate.

.. option:: --verify-client-cacert=<PATH>

    Path  to file  that contains  CA certificates  to verify
    client certificate.  The file must be in PEM format.  It
    can contain multiple certificates.

.. option:: --verify-client-tolerate-expired

    Accept  expired  client  certificate.   Operator  should
    handle  the expired  client  certificate  by some  means
    (e.g.,  mruby  script).   Otherwise, this  option  might
    cause a security risk.

.. option:: --client-private-key-file=<PATH>

    Path to  file that contains  client private key  used in
    backend client authentication.

.. option:: --client-cert-file=<PATH>

    Path to  file that  contains client certificate  used in
    backend client authentication.

.. option:: --tls-min-proto-version=<VER>

    Specify minimum SSL/TLS protocol.   The name matching is
    done in  case-insensitive manner.  The  versions between
    :option:`--tls-min-proto-version` and  :option:`\--tls-max-proto-version` are
    enabled.  If the protocol list advertised by client does
    not  overlap  this range,  you  will  receive the  error
    message "unknown protocol".  If a protocol version lower
    than TLSv1.2 is specified, make sure that the compatible
    ciphers are  included in :option:`--ciphers` option.   The default
    cipher  list  only   includes  ciphers  compatible  with
    TLSv1.2 or above.  The available versions are:
    TLSv1.3, TLSv1.2, TLSv1.1, and TLSv1.0

    Default: ``TLSv1.2``

.. option:: --tls-max-proto-version=<VER>

    Specify maximum SSL/TLS protocol.   The name matching is
    done in  case-insensitive manner.  The  versions between
    :option:`--tls-min-proto-version` and  :option:`\--tls-max-proto-version` are
    enabled.  If the protocol list advertised by client does
    not  overlap  this range,  you  will  receive the  error
    message "unknown protocol".  The available versions are:
    TLSv1.3, TLSv1.2, TLSv1.1, and TLSv1.0

    Default: ``TLSv1.3``

.. option:: --tls-ticket-key-file=<PATH>

    Path to file that contains  random data to construct TLS
    session ticket  parameters.  If aes-128-cbc is  given in
    :option:`--tls-ticket-key-cipher`\, the  file must  contain exactly
    48    bytes.     If     aes-256-cbc    is    given    in
    :option:`--tls-ticket-key-cipher`\, the  file must  contain exactly
    80  bytes.   This  options  can be  used  repeatedly  to
    specify  multiple ticket  parameters.  If  several files
    are given,  only the  first key is  used to  encrypt TLS
    session  tickets.  Other  keys are  accepted but  server
    will  issue new  session  ticket with  first key.   This
    allows  session  key  rotation.  Please  note  that  key
    rotation  does  not  occur automatically.   User  should
    rearrange  files or  change options  values and  restart
    nghttpx gracefully.   If opening  or reading  given file
    fails, all loaded  keys are discarded and  it is treated
    as if none  of this option is given.  If  this option is
    not given or an error  occurred while opening or reading
    a file,  key is  generated every  1 hour  internally and
    they are  valid for  12 hours.   This is  recommended if
    ticket  key sharing  between  nghttpx  instances is  not
    required.

.. option:: --tls-ticket-key-memcached=<HOST>,<PORT>[;tls]

    Specify address  of memcached  server to get  TLS ticket
    keys for  session resumption.   This enables  shared TLS
    ticket key between  multiple nghttpx instances.  nghttpx
    does not set TLS ticket  key to memcached.  The external
    ticket key generator is required.  nghttpx just gets TLS
    ticket  keys  from  memcached, and  use  them,  possibly
    replacing current set  of keys.  It is up  to extern TLS
    ticket  key generator  to rotate  keys frequently.   See
    "TLS SESSION  TICKET RESUMPTION" section in  manual page
    to know the data format in memcached entry.  Optionally,
    memcached  connection  can  be  encrypted  with  TLS  by
    specifying "tls" parameter.

.. option:: --tls-ticket-key-memcached-address-family=(auto|IPv4|IPv6)

    Specify address  family of memcached connections  to get
    TLS ticket keys.  If "auto" is given, both IPv4 and IPv6
    are considered.   If "IPv4" is given,  only IPv4 address
    is considered.  If "IPv6" is given, only IPv6 address is
    considered.

    Default: ``auto``

.. option:: --tls-ticket-key-memcached-interval=<DURATION>

    Set interval to get TLS ticket keys from memcached.

    Default: ``10m``

.. option:: --tls-ticket-key-memcached-max-retry=<N>

    Set  maximum   number  of  consecutive   retries  before
    abandoning TLS ticket key  retrieval.  If this number is
    reached,  the  attempt  is considered  as  failure,  and
    "failure" count  is incremented by 1,  which contributed
    to            the            value            controlled
    :option:`--tls-ticket-key-memcached-max-fail` option.

    Default: ``3``

.. option:: --tls-ticket-key-memcached-max-fail=<N>

    Set  maximum   number  of  consecutive   failure  before
    disabling TLS ticket until next scheduled key retrieval.

    Default: ``2``

.. option:: --tls-ticket-key-cipher=<CIPHER>

    Specify cipher  to encrypt TLS session  ticket.  Specify
    either   aes-128-cbc   or  aes-256-cbc.    By   default,
    aes-128-cbc is used.

.. option:: --tls-ticket-key-memcached-cert-file=<PATH>

    Path to client certificate  for memcached connections to
    get TLS ticket keys.

.. option:: --tls-ticket-key-memcached-private-key-file=<PATH>

    Path to client private  key for memcached connections to
    get TLS ticket keys.

.. option:: --fetch-ocsp-response-file=<PATH>

    Path to  fetch-ocsp-response script file.  It  should be
    absolute path.

    Default: ``/usr/local/share/nghttp2/fetch-ocsp-response``

.. option:: --ocsp-update-interval=<DURATION>

    Set interval to update OCSP response cache.

    Default: ``4h``

.. option:: --ocsp-startup

    Start  accepting connections  after initial  attempts to
    get OCSP responses  finish.  It does not  matter some of
    the  attempts  fail.  This  feature  is  useful if  OCSP
    responses   must    be   available    before   accepting
    connections.

.. option:: --no-verify-ocsp

    nghttpx does not verify OCSP response.

.. option:: --no-ocsp

    Disable OCSP stapling.

.. option:: --tls-session-cache-memcached=<HOST>,<PORT>[;tls]

    Specify  address of  memcached server  to store  session
    cache.   This  enables   shared  session  cache  between
    multiple   nghttpx  instances.    Optionally,  memcached
    connection can be encrypted with TLS by specifying "tls"
    parameter.

.. option:: --tls-session-cache-memcached-address-family=(auto|IPv4|IPv6)

    Specify address family of memcached connections to store
    session cache.  If  "auto" is given, both  IPv4 and IPv6
    are considered.   If "IPv4" is given,  only IPv4 address
    is considered.  If "IPv6" is given, only IPv6 address is
    considered.

    Default: ``auto``

.. option:: --tls-session-cache-memcached-cert-file=<PATH>

    Path to client certificate  for memcached connections to
    store session cache.

.. option:: --tls-session-cache-memcached-private-key-file=<PATH>

    Path to client private  key for memcached connections to
    store session cache.

.. option:: --tls-dyn-rec-warmup-threshold=<SIZE>

    Specify the  threshold size for TLS  dynamic record size
    behaviour.  During  a TLS  session, after  the threshold
    number of bytes  have been written, the  TLS record size
    will be increased to the maximum allowed (16K).  The max
    record size will  continue to be used on  the active TLS
    session.  After  :option:`--tls-dyn-rec-idle-timeout` has elapsed,
    the record size is reduced  to 1300 bytes.  Specify 0 to
    always use  the maximum record size,  regardless of idle
    period.   This  behaviour  applies   to  all  TLS  based
    frontends, and TLS HTTP/2 backends.

    Default: ``1M``

.. option:: --tls-dyn-rec-idle-timeout=<DURATION>

    Specify TLS dynamic record  size behaviour timeout.  See
    :option:`--tls-dyn-rec-warmup-threshold`  for   more  information.
    This behaviour  applies to all TLS  based frontends, and
    TLS HTTP/2 backends.

    Default: ``1s``

.. option:: --no-http2-cipher-black-list

    Allow  black  listed  cipher suite  on  frontend  HTTP/2
    connection.                                          See
    https://tools.ietf.org/html/rfc7540#appendix-A  for  the
    complete HTTP/2 cipher suites black list.

.. option:: --client-no-http2-cipher-black-list

    Allow  black  listed  cipher  suite  on  backend  HTTP/2
    connection.                                          See
    https://tools.ietf.org/html/rfc7540#appendix-A  for  the
    complete HTTP/2 cipher suites black list.

.. option:: --tls-sct-dir=<DIR>

    Specifies the  directory where  \*.sct files  exist.  All
    \*.sct   files   in  <DIR>   are   read,   and  sent   as
    extension_data of  TLS signed_certificate_timestamp (RFC
    6962)  to  client.   These   \*.sct  files  are  for  the
    certificate   specified   in   positional   command-line
    argument <CERT>, or  certificate option in configuration
    file.   For   additional  certificates,   use  :option:`--subcert`
    option.  This option requires OpenSSL >= 1.0.2.

.. option:: --psk-secrets=<PATH>

    Read list of PSK identity and secrets from <PATH>.  This
    is used for frontend connection.  The each line of input
    file  is  formatted  as  <identity>:<hex-secret>,  where
    <identity> is  PSK identity, and <hex-secret>  is secret
    in hex.  An  empty line, and line which  starts with '#'
    are skipped.  The default  enabled cipher list might not
    contain any PSK cipher suite.  In that case, desired PSK
    cipher suites  must be  enabled using  :option:`--ciphers` option.
    The  desired PSK  cipher suite  may be  black listed  by
    HTTP/2.   To  use  those   cipher  suites  with  HTTP/2,
    consider  to  use  :option:`--no-http2-cipher-black-list`  option.
    But be aware its implications.

.. option:: --client-psk-secrets=<PATH>

    Read PSK identity and secrets from <PATH>.  This is used
    for backend connection.  The each  line of input file is
    formatted  as <identity>:<hex-secret>,  where <identity>
    is PSK identity, and <hex-secret>  is secret in hex.  An
    empty line, and line which  starts with '#' are skipped.
    The first identity and  secret pair encountered is used.
    The default  enabled cipher  list might not  contain any
    PSK  cipher suite.   In  that case,  desired PSK  cipher
    suites  must be  enabled using  :option:`--client-ciphers` option.
    The  desired PSK  cipher suite  may be  black listed  by
    HTTP/2.   To  use  those   cipher  suites  with  HTTP/2,
    consider   to  use   :option:`--client-no-http2-cipher-black-list`
    option.  But be aware its implications.

.. option:: --tls-no-postpone-early-data

    By default,  nghttpx postpones forwarding  HTTP requests
    sent in early data, including those sent in partially in
    it, until TLS handshake finishes.  If all backend server
    recognizes "Early-Data" header  field, using this option
    makes nghttpx  not postpone  forwarding request  and get
    full potential of 0-RTT data.

.. option:: --tls-max-early-data=<SIZE>

    Sets  the  maximum  amount  of 0-RTT  data  that  server
    accepts.

    Default: ``16K``


HTTP/2
~~~~~~

.. option:: -c, --frontend-http2-max-concurrent-streams=<N>

    Set the maximum number of  the concurrent streams in one
    frontend HTTP/2 session.

    Default: ``100``

.. option:: --backend-http2-max-concurrent-streams=<N>

    Set the maximum number of  the concurrent streams in one
    backend  HTTP/2 session.   This sets  maximum number  of
    concurrent opened pushed streams.  The maximum number of
    concurrent requests are set by a remote server.

    Default: ``100``

.. option:: --frontend-http2-window-size=<SIZE>

    Sets  the  per-stream  initial  window  size  of  HTTP/2
    frontend connection.

    Default: ``65535``

.. option:: --frontend-http2-connection-window-size=<SIZE>

    Sets the  per-connection window size of  HTTP/2 frontend
    connection.

    Default: ``65535``

.. option:: --backend-http2-window-size=<SIZE>

    Sets  the   initial  window   size  of   HTTP/2  backend
    connection.

    Default: ``65535``

.. option:: --backend-http2-connection-window-size=<SIZE>

    Sets the  per-connection window  size of  HTTP/2 backend
    connection.

    Default: ``2147483647``

.. option:: --http2-no-cookie-crumbling

    Don't crumble cookie header field.

.. option:: --padding=<N>

    Add  at most  <N> bytes  to  a HTTP/2  frame payload  as
    padding.  Specify 0 to  disable padding.  This option is
    meant for debugging purpose  and not intended to enhance
    protocol security.

.. option:: --no-server-push

    Disable HTTP/2 server push.  Server push is supported by
    default mode and HTTP/2  frontend via Link header field.
    It is  also supported if  both frontend and  backend are
    HTTP/2 in default mode.  In  this case, server push from
    backend session is relayed  to frontend, and server push
    via Link header field is also supported.

.. option:: --frontend-http2-optimize-write-buffer-size

    (Experimental) Enable write  buffer size optimization in
    frontend HTTP/2 TLS  connection.  This optimization aims
    to reduce  write buffer  size so  that it  only contains
    bytes  which can  send immediately.   This makes  server
    more responsive to prioritized HTTP/2 stream because the
    buffering  of lower  priority stream  is reduced.   This
    option is only effective on recent Linux platform.

.. option:: --frontend-http2-optimize-window-size

    (Experimental)   Automatically  tune   connection  level
    window size of frontend  HTTP/2 TLS connection.  If this
    feature is  enabled, connection window size  starts with
    the   default  window   size,   65535  bytes.    nghttpx
    automatically  adjusts connection  window size  based on
    TCP receiving  window size.  The maximum  window size is
    capped      by      the     value      specified      by
    :option:`--frontend-http2-connection-window-size`\.     Since   the
    stream is subject to stream level window size, it should
    be adjusted using :option:`--frontend-http2-window-size` option as
    well.   This option  is only  effective on  recent Linux
    platform.

.. option:: --frontend-http2-encoder-dynamic-table-size=<SIZE>

    Specify the maximum dynamic  table size of HPACK encoder
    in the frontend HTTP/2 connection.  The decoder (client)
    specifies  the maximum  dynamic table  size it  accepts.
    Then the negotiated dynamic table size is the minimum of
    this option value and the value which client specified.

    Default: ``4K``

.. option:: --frontend-http2-decoder-dynamic-table-size=<SIZE>

    Specify the maximum dynamic  table size of HPACK decoder
    in the frontend HTTP/2 connection.

    Default: ``4K``

.. option:: --backend-http2-encoder-dynamic-table-size=<SIZE>

    Specify the maximum dynamic  table size of HPACK encoder
    in the backend HTTP/2 connection.  The decoder (backend)
    specifies  the maximum  dynamic table  size it  accepts.
    Then the negotiated dynamic table size is the minimum of
    this option value and the value which backend specified.

    Default: ``4K``

.. option:: --backend-http2-decoder-dynamic-table-size=<SIZE>

    Specify the maximum dynamic  table size of HPACK decoder
    in the backend HTTP/2 connection.

    Default: ``4K``


Mode
~~~~

.. describe:: (default mode)

    
    Accept  HTTP/2,  and  HTTP/1.1 over  SSL/TLS.   "no-tls"
    parameter is  used in  :option:`--frontend` option,  accept HTTP/2
    and HTTP/1.1 over cleartext  TCP.  The incoming HTTP/1.1
    connection  can  be  upgraded  to  HTTP/2  through  HTTP
    Upgrade.

.. option:: -s, --http2-proxy

    Like default mode, but enable forward proxy.  This is so
    called HTTP/2 proxy mode.


Logging
~~~~~~~

.. option:: -L, --log-level=<LEVEL>

    Set the severity  level of log output.   <LEVEL> must be
    one of INFO, NOTICE, WARN, ERROR and FATAL.

    Default: ``NOTICE``

.. option:: --accesslog-file=<PATH>

    Set path to write access log.  To reopen file, send USR1
    signal to nghttpx.

.. option:: --accesslog-syslog

    Send  access log  to syslog.   If this  option is  used,
    :option:`--accesslog-file` option is ignored.

.. option:: --accesslog-format=<FORMAT>

    Specify  format  string  for access  log.   The  default
    format is combined format.   The following variables are
    available:

    * $remote_addr: client IP address.
    * $time_local: local time in Common Log format.
    * $time_iso8601: local time in ISO 8601 format.
    * $request: HTTP request line.
    * $status: HTTP response status code.
    * $body_bytes_sent: the  number of bytes sent  to client
      as response body.
    * $http_<VAR>: value of HTTP  request header <VAR> where
      '_' in <VAR> is replaced with '-'.
    * $remote_port: client  port.
    * $server_port: server port.
    * $request_time: request processing time in seconds with
      milliseconds resolution.
    * $pid: PID of the running process.
    * $alpn: ALPN identifier of the protocol which generates
      the response.   For HTTP/1,  ALPN is  always http/1.1,
      regardless of minor version.
    * $tls_cipher: cipher used for SSL/TLS connection.
    * $tls_client_fingerprint_sha256: SHA-256 fingerprint of
      client certificate.
    * $tls_client_fingerprint_sha1:  SHA-1   fingerprint  of
      client certificate.
    * $tls_client_subject_name:   subject  name   in  client
      certificate.
    * $tls_client_issuer_name:   issuer   name   in   client
      certificate.
    * $tls_client_serial:    serial    number   in    client
      certificate.
    * $tls_protocol: protocol for SSL/TLS connection.
    * $tls_session_id: session ID for SSL/TLS connection.
    * $tls_session_reused:  "r"   if  SSL/TLS   session  was
      reused.  Otherwise, "."
    * $tls_sni: SNI server name for SSL/TLS connection.
    * $backend_host:  backend  host   used  to  fulfill  the
      request.  "-" if backend host is not available.
    * $backend_port:  backend  port   used  to  fulfill  the
      request.  "-" if backend host is not available.
    * $method: HTTP method
    * $path:  Request  path  including query.   For  CONNECT
      request, authority is recorded.
    * $path_without_query:  $path   up  to  the   first  '?'
      character.    For   CONNECT  request,   authority   is
      recorded.
    * $protocol_version:   HTTP  version   (e.g.,  HTTP/1.1,
      HTTP/2)

    The  variable  can  be  enclosed  by  "{"  and  "}"  for
    disambiguation (e.g., ${remote_addr}).


    Default: ``$remote_addr - - [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"``

.. option:: --accesslog-write-early

    Write  access  log  when   response  header  fields  are
    received   from  backend   rather   than  when   request
    transaction finishes.

.. option:: --errorlog-file=<PATH>

    Set path to write error  log.  To reopen file, send USR1
    signal  to nghttpx.   stderr will  be redirected  to the
    error log file unless :option:`--errorlog-syslog` is used.

    Default: ``/dev/stderr``

.. option:: --errorlog-syslog

    Send  error log  to  syslog.  If  this  option is  used,
    :option:`--errorlog-file` option is ignored.

.. option:: --syslog-facility=<FACILITY>

    Set syslog facility to <FACILITY>.

    Default: ``daemon``


HTTP
~~~~

.. option:: --add-x-forwarded-for

    Append  X-Forwarded-For header  field to  the downstream
    request.

.. option:: --strip-incoming-x-forwarded-for

    Strip X-Forwarded-For  header field from  inbound client
    requests.

.. option:: --no-add-x-forwarded-proto

    Don't append  additional X-Forwarded-Proto  header field
    to  the   backend  request.   If  inbound   client  sets
    X-Forwarded-Proto,                                   and
    :option:`--no-strip-incoming-x-forwarded-proto`  option  is  used,
    they are passed to the backend.

.. option:: --no-strip-incoming-x-forwarded-proto

    Don't strip X-Forwarded-Proto  header field from inbound
    client requests.

.. option:: --add-forwarded=<LIST>

    Append RFC  7239 Forwarded header field  with parameters
    specified in comma delimited list <LIST>.  The supported
    parameters  are "by",  "for", "host",  and "proto".   By
    default,  the value  of  "by" and  "for" parameters  are
    obfuscated     string.     See     :option:`--forwarded-by`    and
    :option:`--forwarded-for` options respectively.  Note that nghttpx
    does  not  translate non-standard  X-Forwarded-\*  header
    fields into Forwarded header field, and vice versa.

.. option:: --strip-incoming-forwarded

    Strip  Forwarded   header  field  from   inbound  client
    requests.

.. option:: --forwarded-by=(obfuscated|ip|<VALUE>)

    Specify the parameter value sent out with "by" parameter
    of Forwarded  header field.   If "obfuscated"  is given,
    the string is randomly generated at startup.  If "ip" is
    given,   the  interface   address  of   the  connection,
    including port number, is  sent with "by" parameter.  In
    case of UNIX domain  socket, "localhost" is used instead
    of address and  port.  User can also  specify the static
    obfuscated string.  The limitation is that it must start
    with   "_",  and   only   consists   of  character   set
    [A-Za-z0-9._-], as described in RFC 7239.

    Default: ``obfuscated``

.. option:: --forwarded-for=(obfuscated|ip)

    Specify  the   parameter  value  sent  out   with  "for"
    parameter of Forwarded header field.  If "obfuscated" is
    given, the string is  randomly generated for each client
    connection.  If "ip" is given, the remote client address
    of  the connection,  without port  number, is  sent with
    "for"  parameter.   In  case   of  UNIX  domain  socket,
    "localhost" is used instead of address.

    Default: ``obfuscated``

.. option:: --no-via

    Don't append to  Via header field.  If  Via header field
    is received, it is left unaltered.

.. option:: --no-strip-incoming-early-data

    Don't strip Early-Data header  field from inbound client
    requests.

.. option:: --no-location-rewrite

    Don't  rewrite location  header field  in default  mode.
    When :option:`--http2-proxy`  is used, location header  field will
    not be altered regardless of this option.

.. option:: --host-rewrite

    Rewrite  host and  :authority header  fields in  default
    mode.  When  :option:`--http2-proxy` is  used, these  headers will
    not be altered regardless of this option.

.. option:: --altsvc=<PROTOID,PORT[,HOST,[ORIGIN]]>

    Specify   protocol  ID,   port,  host   and  origin   of
    alternative service.  <HOST>  and <ORIGIN> are optional.
    They  are advertised  in  alt-svc header  field only  in
    HTTP/1.1  frontend.  This  option can  be used  multiple
    times   to   specify  multiple   alternative   services.
    Example: :option:`--altsvc`\=h2,443

.. option:: --add-request-header=<HEADER>

    Specify additional header field to add to request header
    set.  This  option just  appends header field  and won't
    replace anything  already set.  This option  can be used
    several  times   to  specify  multiple   header  fields.
    Example: :option:`--add-request-header`\="foo: bar"

.. option:: --add-response-header=<HEADER>

    Specify  additional  header  field to  add  to  response
    header set.   This option just appends  header field and
    won't replace anything already  set.  This option can be
    used several  times to  specify multiple  header fields.
    Example: :option:`--add-response-header`\="foo: bar"

.. option:: --request-header-field-buffer=<SIZE>

    Set maximum buffer size for incoming HTTP request header
    field list.  This is the sum of header name and value in
    bytes.   If  trailer  fields  exist,  they  are  counted
    towards this number.

    Default: ``64K``

.. option:: --max-request-header-fields=<N>

    Set  maximum  number  of incoming  HTTP  request  header
    fields.   If  trailer  fields exist,  they  are  counted
    towards this number.

    Default: ``100``

.. option:: --response-header-field-buffer=<SIZE>

    Set  maximum  buffer  size for  incoming  HTTP  response
    header field list.   This is the sum of  header name and
    value  in  bytes.  If  trailer  fields  exist, they  are
    counted towards this number.

    Default: ``64K``

.. option:: --max-response-header-fields=<N>

    Set  maximum number  of  incoming  HTTP response  header
    fields.   If  trailer  fields exist,  they  are  counted
    towards this number.

    Default: ``500``

.. option:: --error-page=(<CODE>|*)=<PATH>

    Set file path  to custom error page  served when nghttpx
    originally  generates  HTTP  error status  code  <CODE>.
    <CODE> must be greater than or equal to 400, and at most
    599.  If "\*"  is used instead of <CODE>,  it matches all
    HTTP  status  code.  If  error  status  code comes  from
    backend server, the custom error pages are not used.

.. option:: --server-name=<NAME>

    Change server response header field value to <NAME>.

    Default: ``nghttpx``

.. option:: --no-server-rewrite

    Don't rewrite server header field in default mode.  When
    :option:`--http2-proxy` is used, these headers will not be altered
    regardless of this option.

.. option:: --redirect-https-port=<PORT>

    Specify the port number which appears in Location header
    field  when  redirect  to  HTTPS  URI  is  made  due  to
    "redirect-if-not-tls" parameter in :option:`--backend` option.

    Default: ``443``


API
~~~

.. option:: --api-max-request-body=<SIZE>

    Set the maximum size of request body for API request.

    Default: ``32M``


DNS
~~~

.. option:: --dns-cache-timeout=<DURATION>

    Set duration that cached DNS results remain valid.  Note
    that nghttpx caches the unsuccessful results as well.

    Default: ``10s``

.. option:: --dns-lookup-timeout=<DURATION>

    Set timeout that  DNS server is given to  respond to the
    initial  DNS  query.  For  the  2nd  and later  queries,
    server is  given time based  on this timeout, and  it is
    scaled linearly.

    Default: ``5s``

.. option:: --dns-max-try=<N>

    Set the number of DNS query before nghttpx gives up name
    lookup.

    Default: ``2``

.. option:: --frontend-max-requests=<N>

    The number  of requests that single  frontend connection
    can process.  For HTTP/2, this  is the number of streams
    in  one  HTTP/2 connection.   For  HTTP/1,  this is  the
    number of keep alive requests.  This is hint to nghttpx,
    and it  may allow additional few  requests.  The default
    value is unlimited.


Debug
~~~~~

.. option:: --frontend-http2-dump-request-header=<PATH>

    Dumps request headers received by HTTP/2 frontend to the
    file denoted  in <PATH>.  The  output is done  in HTTP/1
    header field format and each header block is followed by
    an empty line.  This option  is not thread safe and MUST
    NOT be used with option :option:`-n`\<N>, where <N> >= 2.

.. option:: --frontend-http2-dump-response-header=<PATH>

    Dumps response headers sent  from HTTP/2 frontend to the
    file denoted  in <PATH>.  The  output is done  in HTTP/1
    header field format and each header block is followed by
    an empty line.  This option  is not thread safe and MUST
    NOT be used with option :option:`-n`\<N>, where <N> >= 2.

.. option:: -o, --frontend-frame-debug

    Print HTTP/2 frames in  frontend to stderr.  This option
    is  not thread  safe and  MUST NOT  be used  with option
    :option:`-n`\=N, where N >= 2.


Process
~~~~~~~

.. option:: -D, --daemon

    Run in a background.  If :option:`-D` is used, the current working
    directory is changed to '*/*'.

.. option:: --pid-file=<PATH>

    Set path to save PID of this program.

.. option:: --user=<USER>

    Run this program as <USER>.   This option is intended to
    be used to drop root privileges.

.. option:: --single-process

    Run this program in a  single process mode for debugging
    purpose.  Without this option,  nghttpx creates at least
    2  processes:  master  and worker  processes.   If  this
    option is  used, master  and worker  are unified  into a
    single process.  nghttpx still spawns additional process
    if neverbleed is used.  In  the single process mode, the
    signal handling feature is disabled.


Scripting
~~~~~~~~~

.. option:: --mruby-file=<PATH>

    Set mruby script file

.. option:: --ignore-per-pattern-mruby-error

    Ignore mruby compile error  for per-pattern mruby script
    file.  If error  occurred, it is treated as  if no mruby
    file were specified for the pattern.


Misc
~~~~

.. option:: --conf=<PATH>

    Load  configuration  from   <PATH>.   Please  note  that
    nghttpx always  tries to read the  default configuration
    file if :option:`--conf` is not given.

    Default: ``/etc/nghttpx/nghttpx.conf``

.. option:: --include=<PATH>

    Load additional configurations from <PATH>.  File <PATH>
    is  read  when  configuration  parser  encountered  this
    option.  This option can be used multiple times, or even
    recursively.

.. option:: -v, --version

    Print version and exit.

.. option:: -h, --help

    Print this help and exit.



The <SIZE> argument is an integer and an optional unit (e.g., 10K is
10 * 1024).  Units are K, M and G (powers of 1024).

The <DURATION> argument is an integer and an optional unit (e.g., 1s
is 1 second and 500ms is 500 milliseconds).  Units are h, m, s or ms
(hours, minutes, seconds and milliseconds, respectively).  If a unit
is omitted, a second is used as unit.

FILES
-----

*/etc/nghttpx/nghttpx.conf*
  The default configuration file path nghttpx searches at startup.
  The configuration file path can be changed using :option:`--conf`
  option.

  Those lines which are staring ``#`` are treated as comment.

  The option name in the configuration file is the long command-line
  option name with leading ``--`` stripped (e.g., ``frontend``).  Put
  ``=`` between option name and value.  Don't put extra leading or
  trailing spaces.

  When specifying arguments including characters which have special
  meaning to a shell, we usually use quotes so that shell does not
  interpret them.  When writing this configuration file, quotes for
  this purpose must not be used.  For example, specify additional
  request header field, do this:

  .. code-block:: text

    add-request-header=foo: bar

  instead of:

  .. code-block:: text

    add-request-header="foo: bar"

  The options which do not take argument in the command-line *take*
  argument in the configuration file.  Specify ``yes`` as an argument
  (e.g., ``http2-proxy=yes``).  If other string is given, it is
  ignored.

  To specify private key and certificate file which are given as
  positional arguments in command-line, use ``private-key-file`` and
  ``certificate-file``.

  :option:`--conf` option cannot be used in the configuration file and
  will be ignored if specified.

Error log
  Error log is written to stderr by default.  It can be configured
  using :option:`--errorlog-file`.  The format of log message is as
  follows:

  <datetime> <master-pid> <current-pid> <thread-id> <level> (<filename>:<line>) <msg>

  <datetime>
    It is a combination of date and time when the log is written.  It
    is in ISO 8601 format.

  <master-pid>
    It is a master process ID.

  <current-pid>
    It is a process ID which writes this log.

  <thread-id>
    It is a thread ID which writes this log.  It would be unique
    within <current-pid>.

  <filename> and <line>
    They are source file name, and line number which produce this log.

  <msg>
    It is a log message body.

SIGNALS
-------

SIGQUIT
  Shutdown gracefully.  First accept pending connections and stop
  accepting connection.  After all connections are handled, nghttpx
  exits.

SIGHUP
  Reload configuration file given in :option:`--conf`.

SIGUSR1
  Reopen log files.

SIGUSR2

  Fork and execute nghttpx.  It will execute the binary in the same
  path with same command-line arguments and environment variables.  As
  of nghttpx version 1.20.0, the new master process sends SIGQUIT to
  the original master process when it is ready to serve requests.  For
  the earlier versions of nghttpx, user has to send SIGQUIT to the
  original master process.

  The difference between SIGUSR2 (+ SIGQUIT) and SIGHUP is that former
  is usually used to execute new binary, and the master process is
  newly spawned.  On the other hand, the latter just reloads
  configuration file, and the same master process continues to exist.

.. note::

  nghttpx consists of multiple processes: one process for processing
  these signals, and another one for processing requests.  The former
  spawns the latter.  The former is called master process, and the
  latter is called worker process.  If neverbleed is enabled, the
  worker process spawns neverbleed daemon process which does RSA key
  processing.  The above signal must be sent to the master process.
  If the other processes received one of them, it is ignored.  This
  behaviour of these processes may change in the future release.  In
  other words, in the future release, the processes other than master
  process may terminate upon the reception of these signals.
  Therefore these signals should not be sent to the processes other
  than master process.

SERVER PUSH
-----------

nghttpx supports HTTP/2 server push in default mode with Link header
field.  nghttpx looks for Link header field (`RFC 5988
<http://tools.ietf.org/html/rfc5988>`_) in response headers from
backend server and extracts URI-reference with parameter
``rel=preload`` (see `preload
<http://w3c.github.io/preload/#interoperability-with-http-link-header>`_)
and pushes those URIs to the frontend client. Here is a sample Link
header field to initiate server push:

.. code-block:: text

  Link: </fonts/font.woff>; rel=preload
  Link: </css/theme.css>; rel=preload

Currently, the following restriction is applied for server push:

1. The associated stream must have method "GET" or "POST".  The
   associated stream's status code must be 200.

This limitation may be loosened in the future release.

nghttpx also supports server push if both frontend and backend are
HTTP/2 in default mode.  In this case, in addition to server push via
Link header field, server push from backend is forwarded to frontend
HTTP/2 session.

HTTP/2 server push will be disabled if :option:`--http2-proxy` is
used.

UNIX DOMAIN SOCKET
------------------

nghttpx supports UNIX domain socket with a filename for both frontend
and backend connections.

Please note that current nghttpx implementation does not delete a
socket with a filename.  And on start up, if nghttpx detects that the
specified socket already exists in the file system, nghttpx first
deletes it.  However, if SIGUSR2 is used to execute new binary and
both old and new configurations use same filename, new binary does not
delete the socket and continues to use it.

OCSP STAPLING
-------------

OCSP query is done using external Python script
``fetch-ocsp-response``, which has been originally developed in Perl
as part of h2o project (https://github.com/h2o/h2o), and was
translated into Python.

The script file is usually installed under
``$(prefix)/share/nghttp2/`` directory.  The actual path to script can
be customized using :option:`--fetch-ocsp-response-file` option.

If OCSP query is failed, previous OCSP response, if any, is continued
to be used.

:option:`--fetch-ocsp-response-file` option provides wide range of
possibility to manage OCSP response.  It can take an arbitrary script
or executable.  The requirement is that it supports the command-line
interface of ``fetch-ocsp-response`` script, and it must return a
valid DER encoded OCSP response on success.  It must return exit code
0 on success, and 75 for temporary error, and the other error code for
generic failure.  For large cluster of servers, it is not efficient
for each server to perform OCSP query using ``fetch-ocsp-response``.
Instead, you can retrieve OCSP response in some way, and store it in a
disk or a shared database.  Then specify a program in
:option:`--fetch-ocsp-response-file` to fetch it from those stores.
This could provide a way to share the OCSP response between fleet of
servers, and also any OCSP query strategy can be applied which may be
beyond the ability of nghttpx itself or ``fetch-ocsp-response``
script.

TLS SESSION RESUMPTION
----------------------

nghttpx supports TLS session resumption through both session ID and
session ticket.

SESSION ID RESUMPTION
~~~~~~~~~~~~~~~~~~~~~

By default, session ID is shared by all worker threads.

If :option:`--tls-session-cache-memcached` is given, nghttpx will
insert serialized session data to memcached with
``nghttpx:tls-session-cache:`` + lowercase hex string of session ID
as a memcached entry key, with expiry time 12 hours.  Session timeout
is set to 12 hours.

By default, connections to memcached server are not encrypted.  To
enable encryption, use ``tls`` keyword in
:option:`--tls-session-cache-memcached` option.

TLS SESSION TICKET RESUMPTION
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, session ticket is shared by all worker threads.  The
automatic key rotation is also enabled by default.  Every an hour, new
encryption key is generated, and previous encryption key becomes
decryption only key.  We set session timeout to 12 hours, and thus we
keep at most 12 keys.

If :option:`--tls-ticket-key-memcached` is given, encryption keys are
retrieved from memcached.  nghttpx just reads keys from memcached; one
has to deploy key generator program to update keys frequently (e.g.,
every 1 hour).  The example key generator tlsticketupdate.go is
available under contrib directory in nghttp2 archive.  The memcached
entry key is ``nghttpx:tls-ticket-key``.  The data format stored in
memcached is the binary format described below:

.. code-block:: text

    +--------------+-------+----------------+
    | VERSION (4)  |LEN (2)|KEY(48 or 80) ...
    +--------------+-------+----------------+
                   ^                        |
		   |                        |
		   +------------------------+
                   (LEN, KEY) pair can be repeated

All numbers in the above figure is bytes.  All integer fields are
network byte order.

First 4 bytes integer VERSION field, which must be 1.  The 2 bytes
integer LEN field gives the length of following KEY field, which
contains key.  If :option:`--tls-ticket-key-cipher`\=aes-128-cbc is
used, LEN must be 48.  If
:option:`--tls-ticket-key-cipher`\=aes-256-cbc is used, LEN must be
80.  LEN and KEY pair can be repeated multiple times to store multiple
keys.  The key appeared first is used as encryption key.  All the
remaining keys are used as decryption only.

By default, connections to memcached server are not encrypted.  To
enable encryption, use ``tls`` keyword in
:option:`--tls-ticket-key-memcached` option.

If :option:`--tls-ticket-key-file` is given, encryption key is read
from the given file.  In this case, nghttpx does not rotate key
automatically.  To rotate key, one has to restart nghttpx (see
SIGNALS).

CERTIFICATE TRANSPARENCY
------------------------

nghttpx supports TLS ``signed_certificate_timestamp`` extension (`RFC
6962 <https://tools.ietf.org/html/rfc6962>`_).  The relevant options
are :option:`--tls-sct-dir` and ``sct-dir`` parameter in
:option:`--subcert`.  They takes a directory, and nghttpx reads all
files whose extension is ``.sct`` under the directory.  The ``*.sct``
files are encoded as ``SignedCertificateTimestamp`` struct described
in `section 3.2 of RFC 69662
<https://tools.ietf.org/html/rfc6962#section-3.2>`_.  This format is
the same one used by `nginx-ct
<https://github.com/grahamedgecombe/nginx-ct>`_ and `mod_ssl_ct
<https://httpd.apache.org/docs/trunk/mod/mod_ssl_ct.html>`_.
`ct-submit <https://github.com/grahamedgecombe/ct-submit>`_ can be
used to submit certificates to log servers, and obtain the
``SignedCertificateTimestamp`` struct which can be used with nghttpx.

MRUBY SCRIPTING
---------------

.. warning::

  The current mruby extension API is experimental and not frozen.  The
  API is subject to change in the future release.

.. warning::

  Almost all string value returned from method, or attribute is a
  fresh new mruby string, which involves memory allocation, and
  copies.  Therefore, it is strongly recommended to store a return
  value in a local variable, and use it, instead of calling method or
  accessing attribute repeatedly.

nghttpx allows users to extend its capability using mruby scripts.
nghttpx has 2 hook points to execute mruby script: request phase and
response phase.  The request phase hook is invoked after all request
header fields are received from client.  The response phase hook is
invoked after all response header fields are received from backend
server.  These hooks allows users to modify header fields, or common
HTTP variables, like authority or request path, and even return custom
response without forwarding request to backend servers.

There are 2 levels of mruby script invocations: global and
per-pattern.  The global mruby script is set by :option:`--mruby-file`
option and is called for all requests.  The per-pattern mruby script
is set by "mruby" parameter in :option:`-b` option.  It is invoked for
a request which matches the particular pattern.  The order of hook
invocation is: global request phase hook, per-pattern request phase
hook, per-pattern response phase hook, and finally global response
phase hook.  If a hook returns a response, any later hooks are not
invoked.  The global request hook is invoked before the pattern
matching is made and changing request path may affect the pattern
matching.

Please note that request and response hooks of per-pattern mruby
script for a single request might not come from the same script.  This
might happen after a request hook is executed, backend failed for some
reason, and at the same time, backend configuration is replaced by API
request, and then the request uses new configuration on retry.  The
response hook from new configuration, if it is specified, will be
invoked.

The all mruby script will be evaluated once per thread on startup, and
it must instantiate object and evaluate it as the return value (e.g.,
``App.new``).  This object is called app object.  If app object
defines ``on_req`` method, it is called with :rb:class:`Nghttpx::Env`
object on request hook.  Similarly, if app object defines ``on_resp``
method, it is called with :rb:class:`Nghttpx::Env` object on response
hook.  For each method invocation, user can can access
:rb:class:`Nghttpx::Request` and :rb:class:`Nghttpx::Response` objects
via :rb:attr:`Nghttpx::Env#req` and :rb:attr:`Nghttpx::Env#resp`
respectively.

.. rb:module:: Nghttpx

.. rb:const:: REQUEST_PHASE

    Constant to represent request phase.

.. rb:const:: RESPONSE_PHASE

    Constant to represent response phase.

.. rb:class:: Env

    Object to represent current request specific context.

    .. rb:attr_reader:: req

        Return :rb:class:`Request` object.

    .. rb:attr_reader:: resp

        Return :rb:class:`Response` object.

    .. rb:attr_reader:: ctx

        Return Ruby hash object.  It persists until request finishes.
        So values set in request phase hook can be retrieved in
        response phase hook.

    .. rb:attr_reader:: phase

        Return the current phase.

    .. rb:attr_reader:: remote_addr

        Return IP address of a remote client.  If connection is made
        via UNIX domain socket, this returns the string "localhost".

    .. rb:attr_reader:: server_addr

        Return address of server that accepted the connection.  This
	is a string which specified in :option:`--frontend` option,
	excluding port number, and not a resolved IP address.  For
	UNIX domain socket, this is a path to UNIX domain socket.

    .. rb:attr_reader:: server_port

        Return port number of the server frontend which accepted the
        connection from client.

    .. rb:attr_reader:: tls_used

        Return true if TLS is used on the connection.

    .. rb:attr_reader:: tls_sni

        Return the TLS SNI value which client sent in this connection.

    .. rb:attr_reader:: tls_client_fingerprint_sha256

        Return the SHA-256 fingerprint of a client certificate.

    .. rb:attr_reader:: tls_client_fingerprint_sha1

        Return the SHA-1 fingerprint of a client certificate.

    .. rb:attr_reader:: tls_client_issuer_name

        Return the issuer name of a client certificate.

    .. rb:attr_reader:: tls_client_subject_name

        Return the subject name of a client certificate.

    .. rb:attr_reader:: tls_client_serial

        Return the serial number of a client certificate.

    .. rb:attr_reader:: tls_client_not_before

        Return the start date of a client certificate in seconds since
        the epoch.

    .. rb:attr_reader:: tls_client_not_after

        Return the end date of a client certificate in seconds since
        the epoch.

    .. rb:attr_reader:: tls_cipher

        Return a TLS cipher negotiated in this connection.

    .. rb:attr_reader:: tls_protocol

        Return a TLS protocol version negotiated in this connection.

    .. rb:attr_reader:: tls_session_id

        Return a session ID for this connection in hex string.

    .. rb:attr_reader:: tls_session_reused

        Return true if, and only if a SSL/TLS session is reused.

    .. rb:attr_reader:: alpn

        Return ALPN identifier negotiated in this connection.

    .. rb:attr_reader:: tls_handshake_finished

        Return true if SSL/TLS handshake has finished.  If it returns
        false in the request phase hook, the request is received in
        TLSv1.3 early data (0-RTT) and might be vulnerable to the
        replay attack.  nghttpx will send Early-Data header field to
        backend servers to indicate this.

.. rb:class:: Request

    Object to represent request from client.  The modification to
    Request object is allowed only in request phase hook.

    .. rb:attr_reader:: http_version_major

        Return HTTP major version.

    .. rb:attr_reader:: http_version_minor

        Return HTTP minor version.

    .. rb:attr_accessor:: method

        HTTP method.  On assignment, copy of given value is assigned.
        We don't accept arbitrary method name.  We will document them
        later, but well known methods, like GET, PUT and POST, are all
        supported.

    .. rb:attr_accessor:: authority

        Authority (i.e., example.org), including optional port
        component .  On assignment, copy of given value is assigned.

    .. rb:attr_accessor:: scheme

        Scheme (i.e., http, https).  On assignment, copy of given
        value is assigned.

    .. rb:attr_accessor:: path

        Request path, including query component (i.e., /index.html).
        On assignment, copy of given value is assigned.  The path does
        not include authority component of URI.  This may include
        query component.  nghttpx makes certain normalization for
        path.  It decodes percent-encoding for unreserved characters
        (see https://tools.ietf.org/html/rfc3986#section-2.3), and
        resolves ".." and ".".  But it may leave characters which
        should be percent-encoded as is. So be careful when comparing
        path against desired string.

    .. rb:attr_reader:: headers

        Return Ruby hash containing copy of request header fields.
        Changing values in returned hash does not change request
        header fields actually used in request processing.  Use
        :rb:meth:`Nghttpx::Request#add_header` or
        :rb:meth:`Nghttpx::Request#set_header` to change request
        header fields.

    .. rb:method:: add_header(key, value)

        Add header entry associated with key.  The value can be single
        string or array of string.  It does not replace any existing
        values associated with key.

    .. rb:method:: set_header(key, value)

        Set header entry associated with key.  The value can be single
        string or array of string.  It replaces any existing values
        associated with key.

    .. rb:method:: clear_headers

        Clear all existing request header fields.

    .. rb:method:: push(uri)

        Initiate to push resource identified by *uri*.  Only HTTP/2
        protocol supports this feature.  For the other protocols, this
        method is noop.  *uri* can be absolute URI, absolute path or
        relative path to the current request.  For absolute or
        relative path, scheme and authority are inherited from the
        current request.  Currently, method is always GET.  nghttpx
        will issue request to backend servers to fulfill this request.
        The request and response phase hooks will be called for pushed
        resource as well.

.. rb:class:: Response

    Object to represent response from backend server.

    .. rb:attr_reader:: http_version_major

        Return HTTP major version.

    .. rb:attr_reader:: http_version_minor

        Return HTTP minor version.

    .. rb:attr_accessor:: status

        HTTP status code.  It must be in the range [200, 999],
        inclusive.  The non-final status code is not supported in
        mruby scripting at the moment.

    .. rb:attr_reader:: headers

        Return Ruby hash containing copy of response header fields.
        Changing values in returned hash does not change response
        header fields actually used in response processing.  Use
        :rb:meth:`Nghttpx::Response#add_header` or
        :rb:meth:`Nghttpx::Response#set_header` to change response
        header fields.

    .. rb:method:: add_header(key, value)

        Add header entry associated with key.  The value can be single
        string or array of string.  It does not replace any existing
        values associated with key.

    .. rb:method:: set_header(key, value)

        Set header entry associated with key.  The value can be single
        string or array of string.  It replaces any existing values
        associated with key.

    .. rb:method:: clear_headers

        Clear all existing response header fields.

    .. rb:method:: return(body)

        Return custom response *body* to a client.  When this method
        is called in request phase hook, the request is not forwarded
        to the backend, and response phase hook for this request will
        not be invoked.  When this method is called in response phase
        hook, response from backend server is canceled and discarded.
        The status code and response header fields should be set
        before using this method.  To set status code, use
        :rb:attr:`Nghttpx::Response#status`.  If status code is not
        set, 200 is used.  To set response header fields,
        :rb:meth:`Nghttpx::Response#add_header` and
        :rb:meth:`Nghttpx::Response#set_header`.  When this method is
        invoked in response phase hook, the response headers are
        filled with the ones received from backend server.  To send
        completely custom header fields, first call
        :rb:meth:`Nghttpx::Response#clear_headers` to erase all
        existing header fields, and then add required header fields.
        It is an error to call this method twice for a given request.

    .. rb:method:: send_info(status, headers)

        Send non-final (informational) response to a client.  *status*
        must be in the range [100, 199], inclusive.  *headers* is a
        hash containing response header fields.  Its key must be a
        string, and the associated value must be either string or
        array of strings.  Since this is not a final response, even if
        this method is invoked, request is still forwarded to a
        backend unless :rb:meth:`Nghttpx::Response#return` is called.
        This method can be called multiple times.  It cannot be called
        after :rb:meth:`Nghttpx::Response#return` is called.

MRUBY EXAMPLES
~~~~~~~~~~~~~~

Modify request path:

.. code-block:: ruby

    class App
      def on_req(env)
        env.req.path = "/apps#{env.req.path}"
      end
    end

    App.new

Don't forget to instantiate and evaluate object at the last line.

Restrict permission of viewing a content to a specific client
addresses:

.. code-block:: ruby

    class App
      def on_req(env)
        allowed_clients = ["127.0.0.1", "::1"]

        if env.req.path.start_with?("/log/") &&
           !allowed_clients.include?(env.remote_addr) then
          env.resp.status = 404
          env.resp.return "permission denied"
        end
      end
    end

    App.new

API ENDPOINTS
-------------

nghttpx exposes API endpoints to manipulate it via HTTP based API.  By
default, API endpoint is disabled.  To enable it, add a dedicated
frontend for API using :option:`--frontend` option with "api"
parameter.  All requests which come from this frontend address, will
be treated as API request.

The response is normally JSON dictionary, and at least includes the
following keys:

status
  The status of the request processing.  The following values are
  defined:

  Success
    The request was successful.

  Failure
    The request was failed.  No change has been made.

code
  HTTP status code

Additionally, depending on the API endpoint, ``data`` key may be
present, and its value contains the API endpoint specific data.

We wrote "normally", since nghttpx may return ordinal HTML response in
some cases where the error has occurred before reaching API endpoint
(e.g., header field is too large).

The following section describes available API endpoints.

POST /api/v1beta1/backendconfig
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This API replaces the current backend server settings with the
requested ones.  The request method should be POST, but PUT is also
acceptable.  The request body must be nghttpx configuration file
format.  For configuration file format, see `FILES`_ section.  The
line separator inside the request body must be single LF (0x0A).
Currently, only :option:`backend <--backend>` option is parsed, the
others are simply ignored.  The semantics of this API is replace the
current backend with the backend options in request body.  Describe
the desired set of backend severs, and nghttpx makes it happen.  If
there is no :option:`backend <--backend>` option is found in request
body, the current set of backend is replaced with the :option:`backend
<--backend>` option's default value, which is ``127.0.0.1,80``.

The replacement is done instantly without breaking existing
connections or requests.  It also avoids any process creation as is
the case with hot swapping with signals.

The one limitation is that only numeric IP address is allowed in
:option:`backend <--backend>` in request body unless "dns" parameter
is used while non numeric hostname is allowed in command-line or
configuration file is read using :option:`--conf`.

GET /api/v1beta1/configrevision
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This API returns configuration revision of the current nghttpx.  The
configuration revision is opaque string, and it changes after each
reloading by SIGHUP.  With this API, an external application knows
that whether nghttpx has finished reloading its configuration by
comparing the configuration revisions between before and after
reloading.  It is recommended to disable persistent (keep-alive)
connection for this purpose in order to avoid to send a request using
the reused connection which may bound to an old process.

This API returns response including ``data`` key.  Its value is JSON
object, and it contains at least the following key:

configRevision
  The configuration revision of the current nghttpx


SEE ALSO
--------

:manpage:`nghttp(1)`, :manpage:`nghttpd(1)`, :manpage:`h2load(1)`