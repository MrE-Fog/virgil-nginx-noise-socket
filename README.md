</b># virgil-nginx-noise-socket
Nginx module that implements Noise Socket Protocol by using Virgil Security infrastructure.

## Features ##

 - Own context in the Nginx server providing a functionality of TCP of a proxy.
 - Protection of traffic by means of [Noise Protocol](http://noiseprotocol.org/).
 - At the moment only `Noise_XX_25519_AESGCM_BLAKE2b` noise protocol pattern is implemented.

## Building of Nginx with the virgil-nginx-noise-socket module:

#### list of dependences

* Autoconf
* Automake

for Centos 7:
* pcre
* pcre-devel
* pcre2
* pcre2-devel
* openssl-devel
* flex
* bison

#### Building

1. The virgil-nginx-noise-socket module is tested with nginx-1.12.1.
2. Set [Noise-C](https://github.com/rweather/noise-c) library is necessary for building of the server.
* How to build `Noise-C` it is described in [Noise-C Documentation](http://rweather.github.io/noise-c/index.html).
* Installation of library `Noise-C` in system:

```bash
	$ make install
```
###### For option of use as a crypto backend of libsodium:
*  To take stable release of libsodium and to build it: 
  ```bash
    $ git clone https://github.com/jedisct1/libsodium.git -b stable
    $ ./configure
    $ make && make check
    $ sudo make install
  ```
  * To build the ´Noise-C´ library with option:
  
  ```bash
    $ autoreconf -i
    $ ./configure --with-openssl --with-libsodium
    $ make
    $ make check
 ``` 
  * Make sure that the list of the required libraries contains libsodium in the `virgil-nginx-noise-socket/config` file (a line 37, ngx_module_libs="... - lsodium"
  
 2. The example of a script for build of the nginx server with the module is located in `virgil-nginx-noise-socket/example/nginx_configure.sh`.
 
 3. The example of a test configuration of the server is located in `virgil-nginx-noise-socket/example/nginx.conf`. The configuration realizes a functionality of reverse proxy and a backend server working in one copy of nginx launched by the local machine. The configuration works as follows:

<table align = "center">
	<tr>
    <td>https://localhost/<td>-></td>
    	<td>internal redirect to `noise_socket` context<td>-></td>
    	<td>proxy to backend over noise socket<td>---></td>
	</tr>
    <tr>
    	<td>---></td>
        <td>`noise_socket` context on the backend server<td>-></td>
    	<td>internal redirect to http context<td>-></td>
    	<td>access to the static page index.html "Welcome to nginx!"</td>
    </tr>
    <tr>
    </tr>
</table>

4. For operation of the `Noise Protocol` files of private keys are necessary for noise initiator(client) and noise responder (server). Keys are generated by means of the test utility of `echo-keygen` from library`Noise-C` (`noise-c/examples/echo/echo-keygen`). Use of the utility is described in [noise-c example echo](http://rweather.github.io/noise-c/example_echo.html). Examples of files of the generated keys `virgil-nginx-noise-socket/example/server_key_25519` and `virgil-nginx-noise-socket/example/client_key_25519`. 

## Basic directives ##

```nginx
Syntax: 	noise_socket { ... }
Default: 	—
Context: 	main
```

Provides the configuration file context in which the noise socket server directives are specified. 

```nginx
Syntax: 	server { ... } 
Default: 	— 
Context: 	noise_socket
```
Sets the configuration for a server. 

```nginx
Syntax: 	listen address:port [noise] [udp] [backlog=number] [ipv6only=on|off] [reuseport] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];
Default: 	—
Context: 	server
```

The `[noise]` parameter allows specifying that all connections accepted on this port should work in noise socket mode. Defines that this socket is used as noise responder (server). Remaining parameters are similar to the parameters described for the directive of [listen](http://nginx.org/en/docs/stream/ngx_stream_core_module.html#listen)  of the ngx_stream_core_module module.

```nginx
Syntax: 	preread_buffer_size size;
Default: 	preread_buffer_size 65517;
Context: 	noise_socket, server
```
Specifies a size of the preread buffer for server.

```nginx
Syntax: 	server_private_key_file file;
Default: 	—
Context: 	noise_socket, server
```

Specifies a file with the secret key in the format of the simple sequence of bytes for the given noise responder (server). 

```nginx
Syntax: 	client_private_key_file file;
Default: 	—
Context: 	noise_socket, server
```

Specifies a file with the secret key in the format of the simple sequence of bytes for the given noise initiator(client). 

```nginx
Syntax: 	proxy_noise on | off;
Default: 	proxy_noise off;
Context: 	noise_socket, server
```
Enables the noise socket protocol for connections to a proxied server. 

```nginx
Syntax: 	block_buffer_size size;
Default: 	block_buffer_size 65517;
Context: 	noise_socket, server
```

Sets the size of the buffer used for reading data from the proxied server. Also sets the size of the buffer used for reading data from the client.  Value by default is the maximum size of the payload determined in the specification [The Noise Protocol Framework](http://noiseprotocol.org/noise.html). This parameter determines the buffer size for noise initiator(client) and noise responder (server).

```nginx
Syntax: 	noise_handshake_timeout time;
Default: 	noise_handshake_timeout 60s;
Context: 	noise_socket, server
```
Specifies a timeout for the `Noise Protocol` handshake to complete.

#### The module  supports also following directives:

`proxy_pass`, `proxy_bind`, `proxy_connect_timeout`, `proxy_timeout`, `proxy_upload_rate`, `proxy_download_rate`, `proxy_responses`, `proxy_next_upstream`, `proxy_next_upstream_tries`, `proxy_next_upstream_timeout`.<br />
The description of directives same, as for the [ngx_stream_proxy_module module](http://nginx.org/en/docs/stream/ngx_stream_proxy_module.html) only in `noise_socket` context.<br />
`resolver`, `resolver_timeout`, `preread_timeout`, `tcp_nodelay`<br />
The description of directives same, as for the [ngx_stream_core_module module](http://nginx.org/en/docs/stream/ngx_stream_core_module.html) only in `noise_socket` context.<br />

## Keepalive connection configuring

For setup of saving a noise socket session it is possible to use the following directives of nginx for frontend server http: 
```nginx
http {
	...
    proxy_http_version 1.1;
    keepalive_requests 10;
    keepalive_timeout 50s;
	...
	server {
      ...
      location {
      	...
      	proxy_set_header Connection keep-alive;
        ...
      }
    }
    ustream name {
    	....
    	keepalive 1;
        ....
    }
}
    
```
Directives are designated by comments "###For the noise socket connection keepalive setup...###" in the file of a test configuration `virgil-nginx-noise-socket/example/nginx.conf`
