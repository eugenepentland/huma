---
description: Level up your API with tests & code coverage using built-in test utilities.
---

# Writing Tests

Huma provides a number of helpers for testing your API. The most important is the [`humatest`](https://pkg.go.dev/github.com/eugenepentland/huma/v2/humatest) package, which allows you to run a test server and make requests against it.

## Testable Code

First, modify the service code to make it easier to test, by moving the operation registration code out of the `main` function:

```go title="main.go" linenums="1" hl_lines="39 62 71"
package main

import (
	"context"
	"fmt"
	"net/http"

	"github.com/eugenepentland/huma/v2"
	"github.com/eugenepentland/huma/v2/adapters/humachi"
	"github.com/go-chi/chi/v5"
)

// Options for the CLI.
type Options struct {
	Port int `help:"Port to listen on" short:"p" default:"8888"`
}

// GreetingInput represents the greeting operation request.
type GreetingInput struct {
	Name string `path:"name" maxLength:"30" example:"world" doc:"Name to greet"`
}

// GreetingOutput represents the greeting operation response.
type GreetingOutput struct {
	Body struct {
		Message string `json:"message" example:"Hello, world!" doc:"Greeting message"`
	}
}

// ReviewInput represents the review operation request.
type ReviewInput struct {
	Body struct {
		Author  string `json:"author" maxLength:"10" doc:"Author of the review"`
		Rating  int    `json:"rating" minimum:"1" maximum:"5" doc:"Rating from 1 to 5"`
		Message string `json:"message,omitempty" maxLength:"100" doc:"Review message"`
	}
}

func addRoutes(api huma.API) {
	// Register GET /greeting/{name}
	huma.Register(api, huma.Operation{
		OperationID: "get-greeting",
		Summary:     "Get a greeting",
		Method:      http.MethodGet,
		Path:        "/greeting/{name}",
	}, func(ctx context.Context, input *GreetingInput) (*GreetingOutput, error) {
		resp := &GreetingOutput{}
		resp.Body.Message = fmt.Sprintf("Hello, %s!", input.Name)
		return resp, nil
	})

	huma.Register(api, huma.Operation{
		OperationID:   "post-review",
		Summary:       "Post a review",
		Method:        http.MethodPost,
		Path:          "/reviews",
		DefaultStatus: http.StatusCreated,
	}, func(ctx context.Context, i *ReviewInput) (*struct{}, error) {
		// TODO: save review in data store.
		return nil, nil
	})
}

func main() {
	// Create a CLI app which takes a port option.
	cli := huma.NewCLI(func(hooks huma.Hooks, options *Options) {
		// Create a new router & API
		router := chi.NewMux()
		api := humachi.New(router, huma.DefaultConfig("My API", "1.0.0"))

		addRoutes(api)

		// Tell the CLI how to start your server.
		hooks.OnStart(func() {
			fmt.Printf("Starting server on port %d...\n", options.Port)
			http.ListenAndServe(fmt.Sprintf(":%d", options.Port), router)
		})
	})

	// Run the CLI. When passed no commands, it starts the server.
	cli.Run()
}
```

## Writing Tests

Use the [`humatest`](https://pkg.go.dev/github.com/eugenepentland/huma/v2/humatest) package to create a test API and then register your routes against it. Next, make get or post requests against it to test the various user scenarios you have to support:

```go title="main_test.go" linenums="1"
package main

import (
	"strings"
	"testing"

	"github.com/eugenepentland/huma/v2/humatest"
)

func TestGetGreeting(t *testing.T) {
	_, api := humatest.New(t)

	addRoutes(api)

	resp := api.Get("/greeting/world")
	if !strings.Contains(resp.Body.String(), "Hello, world!") {
		t.Fatalf("Unexpected response: %s", resp.Body.String())
	}
}

func TestPutReview(t *testing.T) {
	_, api := humatest.New(t)

	addRoutes(api)

	resp := api.Post("/reviews", map[string]any{
		"author": "daniel",
		"rating": 5,
	})

	if resp.Code != 201 {
		t.Fatalf("Unexpected status code: %d", resp.Code)
	}
}

func TestPutReviewError(t *testing.T) {
	_, api := humatest.New(t)

	addRoutes(api)

	resp := api.Post("/reviews", map[string]any{
		"rating": 10,
	})

	if resp.Code != 422 {
		t.Fatalf("Unexpected status code: %d", resp.Code)
	}
}
```

Now you can run your tests!

```sh title="Terminal"
$ go test -cover
```

## Review

Congratulations! You just learned:

-   How to write tests for your API
-   How to use the `humatest` package to create a test API
-   How to use the `humatest` package to make requests against your API
-   How to run tests and get code coverage

## Dive Deeper

Want to learn more about how testing works? Check these out next:

-   [Operations](../features/operations.md)
-   [Test Utilities](../features/test-utilities.md)
-   [`humatest`](https://pkg.go.dev/github.com/eugenepentland/huma/v2/humatest) reference
