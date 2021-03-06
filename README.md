Erlang mtproto proxy
====================

This part of code was extracted from [@socksy_bot](https://t.me/socksy_bot)

This solution has proven itself and is used as the main server [MTPro.XYZ](https://mtpro.xyz)

Features
--------

* Promoted channels! See `mtproto_proxy_app.src` `tag` option.
* "secure" randomized-packet-size protocol (34-symbol secrets starting with 'dd')
  to prevent detection by DPI
* Secure-only mode (only allow connections with 'dd'-secrets). See `allowed_protocols` option.
* Multiple ports with unique secret and promo tag for each port
* Automatic configuration reload (no need for restarts once per day)
* Most of the configuration options might be updated without service restart
* Very high performance - can handle tens of thousands connections! Scales to all CPU cores.
* Small codebase compared to official one
* A lots of metrics could be exported (optional)

How to start - quick
--------------------

```
sudo apt install erlang-nox erlang-dev build-essential git -y
git clone https://github.com/hookzof/mtproto_proxy
cd mtproto_proxy && cp config/vm.args.example config/prod-vm.args && cp config/sys.config.example config/prod-sys.config
# configure your port, secret, ad_tag. See [Settings](#settings) below.
nano config/prod-sys.config
make && sudo make install && sudo systemctl enable mtproto-proxy && sudo systemctl start mtproto-proxy
```

How to start - detailed
--------------------


### Install deps (ubuntu 18.04)

```
sudo apt install erlang-nox erlang-dev build-essential git -y
```

You need Erlang version 20 or higher! If your version is older, please, check
[Erlang solutions esl-erlang package](https://www.erlang-solutions.com/resources/download.html)
or use [kerl](https://github.com/kerl/kerl).

### Get the code:

```
git clone https://github.com/hookzof/mtproto_proxy
cd mtproto_proxy
```

### Create config file

see [Settings](#settings).

### Build and install

```
make && sudo make install
```

This will:
* install proxy into `/opt/mtp_proxy`
* create a system user
* install systemd service
* create a directory for logs in `/var/log/mtproto-proxy`
* Configure ulimit of max open files and `CAP_NET_BIND_SERVICE` by systemd

### Start and enable start on system start-up

```
sudo systemctl enable mtproto-proxy
sudo systemctl start mtproto-proxy
```

Done! Proxy is up and ready to serve now!

### Stop / uninstall

Stop:

```
sudo systemctl stop mtproto-proxy
```

Uninstall:

```
sudo systemctl stop mtproto-proxy
sudo systemctl disable mtproto-proxy
sudo make uninstall
```

Logs can be found at

```
/var/log/mtproto-proxy/application.log
```

Settings
--------

All possible documanted configuration options could be found
in `src/mtproto_proxy.app.src`. Do not edit this file!

To change configuration, edit `config/prod-sys.config`:

Default port is 443 and default secret is `SECRET`.

Secret key and proxy URL will be printed on start.

The easiest way to update config right now is to edit `config/prod-sys.config`
and then re-install proxy by
```
sudo make uninstall && make && sudo make install
```
There are other ways as well. It's even possible to update configuration options
without service restart / without downtime, but it's a bit trickier.

### Change default port / secret / ad tag

To change default settings, change `mtproto_proxy` section of `prod-sys.config` as:

```
 {mtproto_proxy,
  %% see src/mtproto_proxy.app.src for examples.
  %% DO NOT EDIT src/mtproto_proxy.app.src!!!
  [
   {ports,
    [#{name => mtp_handler1,
       listen_ip => "0.0.0.0",
       port => 443,
       secret => <<"SECRET">>,
       tag => <<"TAG">>}
    ]}
   ]},

 {lager,
<...>
```
Replace `port` / `secret` / `tag` with yours.

### Listen on multiple ports / IPs

You can start proxy on many IP addresses or ports with different secrets/ad tags.
To do so, just add more configs to `ports` section, separated by comma, eg:

```
 {mtproto_proxy,
  %% see src/mtproto_proxy.app.src for examples.
  %% DO NOT EDIT src/mtproto_proxy.app.src!!!
  [
   {ports,
    [#{name => mtp_handler_1,
       listen_ip => "0.0.0.0",
       port => 443,
       secret => <<"SECRET">>,
       tag => <<"TAG">>},
     #{name => mtp_handler_2,
       listen_ip => "0.0.0.0",
       port => 1443,
       secret => <<"SECRET">>,
       tag => <<"TAG">>}
    ]}
   ]},

 {lager,
<...>
```

Each section should have unique `name`!

### Only allow connections with 'dd'-secrets

It might be useful in Iran, where proxies are detected by DPI.
You should disable all protocols other than `mtp_secure` by providing `allowed_protocols` option:

```
  {mtproto_proxy,
   [
    {allowed_protocols, [mtp_secure]},
    {ports,
     [#{name => mtp_handler_1,
      <..>
```


Helpers
-------

Number of connections

```
/opt/mtp_proxy/bin/mtp_proxy eval 'lists:sum([proplists:get_value(all_connections, L) || {_, L} <- ranch:info()]).'
```
