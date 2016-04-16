# KONG - The API Layer

[![Build Status][travis-badge]][travis-url]
[![Coverage Status][coveralls-badge]][coveralls-url]
[![Gitter][gitter-badge]][gitter-url]

Kong is a scalable and customizable API Management Layer built on top of Nginx.

* **[Installation](#installation)**
* **[Documentation](#documentation)**
* **[Usage](#usage)**
* **[Development](#development)**

## Installation

See [INSTALL.md](INSTALL.md) for installation instructions on your platform.

## Usage

Use Kong through the `kong` executable. If you installed Kong via luarocks, then `kong` should be in your `$PATH`.

```bash
$ kong --help
```

## Getting started

Kong will look by default for a configuration file at `/etc/kong/kong.yml`. If you installed Kong from luarocks, you can copy the default configuration from the luarocks tree (`luarocks --help` to print it). Usually:

```bash
cp /usr/local/lib/luarocks/rocks/kong/<kong_version>/conf/kong.yml /etc/kong/kong.yml
```

Edit the configuration to let Kong access your Cassandra cluster.

Let's start Kong:

```bash
$ kong start
```

This should have run the migrations to prepare your Cassandra keyspace, and you should see a success message if Kong has started.

Kong listens on these ports:
- `:8000`: requests proxying
- `:8001`: Kong's configuration API by which you can add APIs and accounts

#### Hello World: Proxying your first API

Let's add [mockbin](http://mockbin.com/) as an API:

```bash
$ curl -i -X POST \
  --url http://localhost:8001/apis/ \
  --data 'name=mockbin&target_url=http://mockbin.com/&public_dns=mockbin.com'
HTTP/1.1 201 Created
```

We used the `8001` port, the one of the configuration API.

And query it through Kong, by using the `8000` port, the one actually proxying requests:

```bash
$ curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: mockbin.com'
HTTP/1.1 200 OK
```

Kong just forwareded our request to `target_url` of mockbin and sent us the response.

#### Consumers and plugins

One of Kong's core principle is its extensibility through [plugins](http://getkong.org/plugins/), which allow you to add features to your APIs.

Let's configure the [keyauth](http://getkong.org/plugins/key-authentication/) plugin to add authentication to your API.

###### 1. Enable the plugin

Make sure it is in the `plugins_available` property of your node's configuration:

```yaml
plugins_available:
  - keyauth
```

This will make Kong load the plugin. If the plugin was not previously marked as available, restart Kong:

```bash
$ kong restart
```

Repeat this step for every node in your cluster.

###### 2. Configure the plugin for an API

To enable this plugin on an API, retrieve the API `id` (as the one of the freshly created mockbin API) and perform a `POST` request with the following parameters:

> **name**: name of the plugin to create a configuration for.
> **api_id**: `id` of the API this plugin will be added to.
> **value.header_names**: `value` is a property of every plugin, representing their configuration. `header_names` will here be a list of headers in which users can pass their API key.

```bash
$ curl -i -X POST \
  --url http://localhost:8001/plugins_configurations/ \
  --data 'name=headerauth&api_id=<api_id>&value.header_names=apikey'
HTTP/1.1 201 Created
...
{
  "api_id": "<api_id>",
  "value": { "header_names":["apikey"], "hide_credentials":false },
  "id": "<id>",
  "enabled": true,
  "name": "headerauth"
}
```

Here, the plugin has been successfully created and enabled. 

If you make the same request against the mockbin API again:

```bash
$ curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: mockbin.com'
HTTP/1.1 403 Forbidden
...
{ "message": "Your authentication credentials are invalid" }
```

This request did not provide a header named `apikey` (as specified by our plugin configuration) and is thus Forbidden. Kong does not proxy the request to the final API.

To authenticate against your API, you now need to create a consumer account and a credential key:

```bash
$ curl -i -X POST \ 
  --url http://localhost:8001/accounts/
  --data ''
HTTP/1.1 201 Created

# Make sure the given account_id matches the freshly created account
$ curl -i -X POST \
  --url http://localhost:8001/keyauth_credentials/
  --data 'key=123456&consumer_id=<consumer_id>'
HTTP/1.1 201 Created
```

That application (which has `123456` as an API key) can now consume APIs having an authentication requirement such as our example. If we make the same query again, but with an `apikey` header containing our credential key:

```bash
$ curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: mockbin.com' \
  --header 'apikey: 123456'
HTTP/1.1 200 OK
```

Your user can now consume this API!

To go further into mastering Kong and its plugins, refer to the complete [documentation](#documentation).

## Documentation

A complete documentation on how to configure and use Kong can be found at: [getkong.org/docs](http://getkong.org/docs). **(coming soon)**

## Development

To develop for Kong, simply run `[sudo] make install` in a clone of this repo. Then run:

```bash
$ make dev
```

This will install development dependencies and create your environment configuration files (`kong_TESTS.yml` and `kong_DEVELOPMENT.yml`).

- Run the tests:

```bash
$ make test-all
```

- Run Kong with the development configuration:

```bash
$ kong start -c kong_DEVELOPMENT.yml
```

#### Makefile

When developing, use the `Makefile` for doing the following operations:

| Name          | Description                                                                                         |
| ------------- | --------------------------------------------------------------------------------------------------- |
| `install`     | Install the Kong luarock globally                                                                   |
| `dev`         | Setup your development environment                                                                  |
| `run`         | Run the `DEVELOPMENT` environment (`kong_DEVELOPMENT.yml`)                                          |
| `seed`        | Seed the `DEVELOPMENT` environment (`kong_DEVELOPMENT.yml`)                                         |
| `drop`        | Drop the `DEVELOPMENT` environment (`kong_DEVELOPMENT.yml`)                                         |
| `lint`        | Lint Lua files in `src/`                                                                            |
| `coverage`    | Run unit tests + coverage report (only unit-tested modules)                                         |
| `test`        | Run the unit tests                                                                                  |
| `test-proxy`  | Run the proxy integration tests                                                                     |
| `test-server` | Run the server integration tests                                                                    |
| `test-api`    | Run the api integration tests                                                                       |
| `test-all`    | Run all unit + integration tests at once                                                            |

[travis-url]: https://travis-ci.org/Mashape/kong
[travis-badge]: https://img.shields.io/travis/Mashape/kong.svg?style=flat
[coveralls-url]: https://coveralls.io/r/Mashape/kong?branch=master
[coveralls-badge]: https://coveralls.io/repos/Mashape/kong/badge.svg?branch=master
[gitter-url]: https://gitter.im/Mashape/kong?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge
[gitter-badge]: https://badges.gitter.im/Join%20Chat.svg
