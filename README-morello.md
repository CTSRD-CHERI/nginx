# Background

This software is an experimental Morello port of nginx v1.24.0.

## Build instructions

nginx is build with GNU make (gmake), first install this build dependency
as follows:

`$ sudo pkg64 install gmake`

Once installed nginx can be built as follows
(see `auto/configure --help` for a complete list of options):
 
```
$ auto/configure \
	--with-cc-opt='-Wno-cheri-provenance' \
	--without-http_geo_module \
	--with-http_ssl_module \
	--with-pcre \
	--with-compat
$ gmake
$ sudo gmake install
```

## Run instructions

Once installed nginx can be started by specifying a configuration file as:
 
`sudo /usr/local/nginx/sbin/nginx -c <conf>`

To stop the server:

`sudo /usr/local/nginx/sbin/nginx -s stop`

Further details ons starting and stopping nginx and writting a configuration
file can be found,
[Starting, Stopping, and Restarting NGINX](https://www.nginx.com/resources/wiki/start/topics/tutorials/commandline/) and
[Creating NGINX Plus and NGINX Configuration Files](https://docs.nginx.com/nginx/admin-guide/basic-functionality/managing-configuration-files/).

## Library compartmentalization

Library compartmentalisation can be enabled for nginx following the
instructions given in `man c18n`.  For example:

`sudo env LD_COMPARTMENT_ENABLE=yes /usr/local/nginx/sbin/nginx -c ...`

nginx should then start running with shared libraries within seperate
compartmentments.

## Testing

### Unit testing

Unit testing has been performed using a corpus of Perl scripts
[nginx-tests](http://hg.nginx.org/nginx-tests). The core HTTP function can be
tested as (this require installation of Perl to provide the `prove` command
line utility):

```
TEST_NGINX_BINARY=/usr/local/nginx/sbin/nginx prove http*
http_absolute_redirect.t .... ok
http_disable_symlinks.t ..... skipped: no disable_symlinks
http_error_page.t ........... ok
http_expect_100_continue.t .. ok
http_header_buffers.t ....... ok
http_headers_multi.t ........ ok
http_host.t ................. ok
http_include.t .............. ok
http_keepalive.t ............ ok
http_keepalive_shutdown.t ... ok
http_listen.t ............... ok
http_listen_wildcard.t ...... skipped: listen on wildcard address
http_location.t ............. ok
http_location_auto.t ........ ok
http_location_win32.t ....... skipped: not win32
http_method.t ............... ok
http_resolver.t ............. ok
http_resolver_aaaa.t ........ ok
http_resolver_cleanup.t ..... ok
http_resolver_cname.t ....... ok
http_resolver_ipv4.t ........ skipped: no resolver ipv4
http_server_name.t .......... ok
http_try_files.t ............ ok
http_uri.t .................. ok
http_variables.t ............ ok
All tests successful.
Files=25, Tests=402, 67 wallclock secs ( 0.20 usr  0.05 sys +  4.80 cusr  1.07 csys =  6.11 CPU)
Result: PASS
```

NOTE: That the `http_header_buffers.t` script requires increasing the
connection pool size to 224.

To run the unit tests with library compartmentalisation
include the `LD_COMPARTMENT_ENABLE` environmental variable:

`LD_COMPARTMENT_ENABLE=yes TEST_NGINX_BINARY=/usr/local/nginx/sbin/nginx prove http*`

### Performance testing

Performance testing is performed using the `wrk` benchmark, as described
in [Testing the Performance of NGINX and NGINX Plus Web Servers](https://www.nginx.com/blog/testing-the-performance-of-nginx-and-nginx-plus-web-servers/).

`./wrk -t 1 -c 50 -d 15s --latency https://192.168.2.2/1kb.bin`

## Notes and Limitations

As with many configure scripts, `-Werror` is enabled. This results in many
ambiguous provenance warnings which (in theses cases) are being promoted
to errors. As these warnings are benign and the changes to the code to quiet
them would be disruptive, the warning should be disabled by passing
`--with-cc-opt='-Wno-cheri-provenenace` to configure.

The `http_geo_module` performs a cast of a pointer difference to a pointer.
This results in a capability misuse as the resulting pointer can't be
dereferenced. 

Adaptations to nginx to the memory-safe CheriABI have been driven by:
compiler warnings and errors, and dynamic testing. Where the compiler
emits a warning or error we are able to rigorously review this and
correct. However, some issues only manifest dynamically (at runtime),
such as invalidation of capabilities by pointer arithmetic,
non-blessed memory copies, or insufficient pointer alignment.
Enhancements such as CHERI UBsan have modestly improved the ability to
identify problems previously only found during dynamic testing. However,
 we are still greatly reliant on dynamic testing. This testing is
constrained by both the completeness of the test suites (which in some
cases provide poor coverage) and the time available within the project
to perform testing. Whilst it is known that errors remain outside the
core http module we are not able to estimate what problems might
remain beyond those resolved in the scope of the project.

## Acknowledgement

This work has been undertaken within DSTL contract
ACC6036483: CHERI-based compartmentalisation for web services on Morello.
