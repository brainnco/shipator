# shipator

[![CI](https://github.com/brainnco/shipator/workflows/CI/badge.svg?branch=main)](https://github.com/brainnco/shipator/actions/workflows/CI.yml?query=branch%3Amain)
[![GitHub release](https://img.shields.io/github/v/release/brainnco/shipator)](https://github.com/brainnco/shipator/releases/latest)
[![Go Report Card](https://goreportcard.com/badge/github.com/brainnco/shipator)](https://goreportcard.com/report/github.com/brainnco/shipator)
[![License](https://img.shields.io/github/license/brainnco/shipator)](https://github.com/brainnco/shipator/blob/main/LICENSE)

Inject environment variables into static files at runtime.

## Background

The common frontend build tools (like [CRA][CRA]) inject the env vars at build time.
This means if we want to deploy the static files in a docker environment,
we need to build a different docker image for each environment, as the url for
the server and other values that usually come from a env var will be inlined in the final bundle.
This behaviour prevent the docker image to be built once and run anywhere - on any
environment (dev, test, staging, prod) - just requiring us to provide the different configuration of each environment.

This project aims to feel this gap by allowing to inject environment variables into
static files at runtime using the [approach mentioned at the `CRA` documentation](https://create-react-app.dev/docs/title-and-meta-tags#injecting-data-from-the-server-into-the-page).

Another option would be serve the application using a [Node.js][Node.js] webserver to inject the
env vars in each request. Shipator takes a different approach by trying to keep working
exclusively with static files in order to leverage high performance webservers like
[ngnix](https://www.nginx.com/) to serve the files while enabling us to inject env vars at runtime.

## How it works

Shipator will replace a placeholder (like `__ENV__`) of a targeted static file with the allowed env vars.

It ships with a official docker image that uses [OpenResty](OpenResty)
(a [ngnix](https://www.nginx.com/) distribution) to serve the files. Whenever the container starts,
it will read all the env vars starting with the prefix defined at `SHIPATOR_PREFIX` (default `REACT_APP`),
and inject the env vars in the place of the placeholder defined at `SHIPATOR_PLACEHOLDER` (default `__ENV__`)
of the target file defined at `SHIPATOR_TARGET` (default `/app/shipator/html/index.html`).
After the replace the ngnix server will be started to serve the static files.

## Docker Usage

Note: The following example assumes that your project was generated by [CRA][CRA]
however all the steps can be applied to different setups given a few minor modifications.

1. Update the `public/index.html` of your project, inserting the `__ENV__` to be
injected at runtime, but fallbacking to the build time env vars that [CRA][CRA]
provides, so it work locally normally.

Insert the following snippet right before the `</head>` closing tag.

```html
<script>
  window.ENV = (function() {
    try {
      return __ENV__;
    } catch (e) {
      return {};
    }
  })();
  window.ENV.REACT_APP_SERVER_URL =
    window.ENV.REACT_APP_SERVER_URL || '%REACT_APP_SERVER_URL%';
</script>
```

2. Update the rest of the project to get the values from this global variable:

```ts
const baseUrl = window.ENV.REACT_APP_SERVER_URL;
```

3. Create a Dockerfile using shipator

```docker
FROM node:12.16.1-alpine as builder
WORKDIR /app
COPY yarn.lock /app/yarn.lock
COPY package.json /app/package.json
RUN yarn install --frozen-lockfile
COPY . /app
RUN yarn build

FROM brainnco/shipator:0.1.0-rc5
COPY --from=builder /app/build /app/shipator/html
```

4. Build the docker image.

```sh
$ docker build --rm -t my-webapp:latest .
```

5. Start the webserver

```
$ docker run --rm -p 3000:80 -e REACT_APP_SERVER_URL=http://my-api.com my-webapp:latest
```

6. Now open your browser at http://localhost:3000.

If you open your dev tools console, and write `window.ENV.REACT_APP_SERVER_URL`,
it will return `http://my-api.com`.

Note: If you use [kubernetes](https://kubernetes.io/) in production the env vars
could be placed in a configmap, with different values for each deployment environment.

## CLI Usage

You can also use the shipator CLI directly. You can download it in the [releases page](https://github.com/brainnco/shipator/releases).

```
Usage
  $ shipator [options] target

Options
  -placeholder string
        Placeholder in the target (default "__ENV__")
  -prefix string
        Prefix of the env vars to inject (default "REACT_APP")
  -version
        Prints current version

Examples
  $ shipator build/index.html
  $ shipator -prefix REACT_APP -placeholder __ENV__ build/index.html
  $ shipator -placeholder __VARS__ build/index.html
  $ shipator -prefix VUE_APP build/index.html
```

## Changelog

See the [changelog](https://github.com/brainnco/shipator/blob/main/CHANGELOG.md).

## Contributing

See the [contributing file](https://github.com/brainnco/shipator/blob/main/CONTRIBUTING.md).

## License

Copyright 2020 brainn.co.

Shipator source code is released under Apache 2 License.

Check [LICENSE](https://github.com/brainnco/shipator/blob/main/LICENSE) file for more information.

[CRA]: https://create-react-app.dev
[Node.js]: https://nodejs.org
[nginx]: https://www.nginx.com
[OpenResty]: https://openresty.org
