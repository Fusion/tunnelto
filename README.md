<p align="center" >
<img width="540px" src="https://repository-images.githubusercontent.com/249120770/7ea6d180-b4ba-11ea-96ab-6c3b987aac9d" align="center"/>
</p>

<p align="center">    
  <a href="https://github.com/agrinman/tunnelto/actions?query=workflow%3A%22Build+and+Release%22"><img src="https://github.com/agrinman/wormhole/workflows/Build%20and%20Release/badge.svg" alt="BuildRelease"></a>
  <a href="https://crates.io/crates/wormhole-tunnel"><img src="https://img.shields.io/crates/v/tunnelto" alt="crate"></a>
  <a href="https://github.com/agrinman/tunnelto/packages/295195"><img src="https://img.shields.io/docker/v/agrinman/wormhole?label=Docker" alt="GitHub Docker Registry"></a> 
  <a href="https://twitter.com/alexgrinman"><img src="https://img.shields.io/twitter/follow/alexgrinman?label=%40AlexGrinman" alt="crate"></a>
</p>

# `tunnelto`
`tunnelto` lets you expose your locally running web server via a public URL.
Written in Rust. Built completely with async-io on top of tokio.

1. [Install](#install)
2. [Usage Instructions](#usage)
3. [Host it yourself](#host-it-yourself)

# Install
## Brew (macOS)
```bash
brew install agrinman/tap/tunnelto
```

## Cargo
```bash
cargo install tunnelto
```

## Everywhere
Or **Download a release for your target OS here**: [tunnelto/releases](https://github.com/agrinman/tunnelto/releases)

# Usage
## Quick Start
```shell script
tunnelto --port 8000
```
The above command opens a tunnel and forwards traffic to `localhost:8000`.

## More Options:
```shell script
tunnelto 0.1.14

USAGE:
    tunnelto [FLAGS] [OPTIONS] [SUBCOMMAND]

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information
    -v, --verbose    A level of verbosity, and can be used multiple times

OPTIONS:
        --dashboard-address <dashboard-address>    Sets the address of the local introspection dashboard
    -k, --key <key>                                Sets an API authentication key to use for this tunnel
        --host <local-host>
            Sets the HOST (i.e. localhost) to forward incoming tunnel traffic to [default: localhost]

    -p, --port <port>
            Sets the port to forward incoming tunnel traffic to on the target host

        --scheme <scheme>
            Sets the SCHEME (i.e. http or https) to forward incoming tunnel traffic to [default: http]

    -s, --subdomain <sub-domain>                   Specify a sub-domain for this tunnel

SUBCOMMANDS:
    help        Prints this message or the help of the given subcommand(s)
    set-auth    Store the API Authentication key
```

# Host it yourself
1. Compile the server for the musl target. See the `musl_build.sh` for a way to do this trivially with Docker!
2. See `Dockerfile` for a simple alpine based image that runs that server binary.
3. Deploy the image where ever you want.

## Testing Locally
```shell script
# Run the Server: xpects TCP traffic on 8080 and control websockets on 5000
ALLOWED_HOSTS="localhost" cargo run --bin tunnelto_server

# Run a local tunnelto client talking to your local tunnelto_server
CTRL_HOST="localhost" CTRL_PORT=5000 CTRL_TLS_OFF=1 cargo run --bin tunnelto -- -p 8000

# Test it out!
# Remember 8080 is our local tunnelto TCP server
curl -H '<subdomain>.localhost' "http://localhost:8080/some_path?with=somequery"
```
See `tunnelto_server/src/config.rs` for the environment variables for configuration.

## Caveats for hosting it yourself
Hello! Chris here.

> It is 2025 and the pull request I submitted 4 years ago was unfortunately never merged. Over the years, people have either posted comments or contacted me directly to express how much that simple code change has helped them, so here it is, feel free to use it.

Basically this change allows you to use your own local **sqlite database** rather than integrate with DynamoDB.

### Building

Debug build:
```shell script
cargo build --no-default-features --features sqlite
```

Release build:
```shell script
cargo build --no-default-features --features sqlite --release
```

### Provisioning
```shell script
ALLOWED_HOSTS="<your subdomain>" \
CTRL_HOST="<server hostname>" \
PORT=<client port> \
CTRL_PORT=<control port> \
tunnelto_server
```

The first time you run this, it will create the sqlite database. Try connecting to the sever from a client:
```shell script
CTRL_HOST="<server hostname>" 
CTRL_PORT=<control port> 
CTRL_TLS_OFF=1 
tunnelto --port <some local port> -s <some irrelevant name> 
-k <the key you wish to authenticate with>
```

You will see, in the server's output, this string:
```
Encrypted key: "<your authentication key, encrypted>"
```

Make a note of the encrypted key value, stop the server, then:

```shell script
cat /proc/sys/kernel/random/uuid # your account id
cat /proc/sys/kernel/random/uuid # your subscription id
sqlite3 tunnelto.db
```

```sql
insert into tunnelto_auth(account_id,auth_key_hash)
    values('<your account id>','<encrypted key>');
insert into tunnelto_record (account_id,subscription_id)
    values('<your account id>', '<your subscription id>')
```

You now have a user in good standing.

### Adding a subdomain (`<xxx.your subdomain>`)
```sql
insert into tunnelto_domains (subdomain,account_id)
    values('<endpoint>', '<your account id>')
```

This endpoint is now **authorized**. You can dynamically instantiate it using a client call (see below)

### Using
Server-side:
```shell script
ALLOWED_HOSTS="<your subdomain>" \
CTRL_HOST="<server hostname>" \
TUNNEL_HOST="<server hostname>" \
PORT=<client port> \
CTRL_PORT=<control port> \
tunnelto_server
```

On the client:
```shell script
CTRL_HOST="<your subdomain>" \
CTRL_PORT=<control port> \
CTRL_TLS_OFF=1 \
tunnelto --port <local port>> -s <endpoint> \
-k <your key>
```

Of course, entering your key every single time can get old fast. That's why alternatively you can first run:
```shell script
tunnelto set-auth --key <your key>
```

then, on demand:
```shell script
CTRL_HOST="<your subdomain>" \
CTRL_PORT=<control port> \
CTRL_TLS_OFF=1 \
tunnelto --port <local port>> -s <endpoint>
```

### Automating the server life cycle
Create a systemd unit; for instance: `/etc/systemd/system/tunnelto_server.service`:
```toml
[Unit]
Description=Run tunnelto service
AssertPathExists=/root/tunnelto/tunnelto_server

[Service]
User=root
Group=root
WorkingDirectory=/root/tunnelto
Environment="ALLOWED_HOSTS=<your subdomain>"
Environment="CTRL_HOST=<server hostname>"
Environment="TUNNEL_HOST=<server hostname>"
Environment="PORT=<client port>"
Environment="CTRL_PORT=<control port>"
Environment="PATH=/root/.asdf/installs/rust/1.45.0/bin:/root/.asdf/shims:/root/.asdf/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/root/.asdf/installs/rust/1.45.0/bin/cargo run --bin tunnelto_server

[Install]
WantedBy=multi-user.target
```

You will need to adapt your `PATH` environment as well as `ExecStart`

Then:
```shell script
sudo systemctl daemon-reload
sudo systemctl start tunnelto_server
sudo systemctl enable tunnelto_server
```
