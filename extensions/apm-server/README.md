# APM Server extension

Adds a container for Elasticsearch APM server. Forwards caught errors and traces to Elasticsearch to enable their
visualisation in Kibana.

## Usage

If you want to include the APM server, run Docker Compose from the root of the repository with an additional command
line argument referencing the `apm-server-compose.yml` file:

```console
$ docker-compose -f docker-compose.yml -f extensions/apm-server/apm-server-compose.yml up
```

```sh
curl -L -O https://raw.githubusercontent.com/elastic/apm-server/7.5/apm-server.docker.yml
```

## Connecting an agent to APM-Server

The most basic configuration to send traces to APM server is to specify the `SERVICE_NAME` and `SERVICE_URL`. Here is an
example Golang Echo configuration:

```go
package main

import (
	echo "github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"
	"net/http"

	"go.elastic.co/apm/module/apmechov4"

)

func main() {
	e := echo.New()
	e.Use(apmechov4.Middleware())


	// Middleware
	e.Use(middleware.Logger())
	e.Use(middleware.Recover())

	e.GET("/", func(c echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})
	e.Logger.Fatal(e.Start(":1323"))
}
```

More configuration settings can be found under the **Configuration** section for each language:
https://www.elastic.co/guide/en/apm/agent/index.html

## Checking connectivity and importing default APM dashboards

From the Kibana Dashboard:

1. `Add APM` button under _Add Data to Kibana_ section
2. Ignore all the install instructions and press `Check APM Server status` button.
3. Press `Check agent status`
4. Press `Load Kibana objects` to get the default dashboards
5. Lastly press the `APM dashboard` to the bottom right.

## See also

[Running APM Server on Docker](https://www.elastic.co/guide/en/apm/server/current/running-on-docker.html)
