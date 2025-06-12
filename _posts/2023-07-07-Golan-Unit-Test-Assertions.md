---
title: "Golang Unit Testing Assertions"
tags:
    - Golang
---

Having being used to the use of assertions for unit testing in languages like Python Ruby and Rust, I was a little surprised to find that the Golang `testing` package didn't provide any assertion support.

Looking at the example below from the [go.dev tutorial](https://go.dev/doc/tutorial/add-a-test), the use of the manual conditional checks and `t.Fatalf()` are the `testing` packages equivalent of an assertion. I can see the extra flexibility that it provides in having control over the set of conditionals used and the choice to treat it as an error or a fatal error but it does seem a little verbose for more simple assertions.

```go
package greetings

import (
    "testing"
)

// TestHelloEmpty calls greetings.Hello with an empty string,
// checking for an error.
func TestHelloEmpty(t *testing.T) {
    msg, err := Hello("")
    if msg != "" || err == nil {
        t.Fatalf(`Hello("") = %q, %v, want "", error`, msg, err)
    }
}
```

The `github.com/stretchr/testify/assert` package does provide the familiar assertion capabilities that I'm used to from other languages, but it does seem odd that these have not been incorporated into the `testing` package itself rather than relying on a 3rd party library.

```go
package greetings

import (
    "github.com/stretchr/testify/assert"
    "testing"
)

// TestHelloEmpty calls greetings.Hello with an empty string,
// checking for an error.
func TestHelloEmpty(t *testing.T) {
    msg, err := Hello("")
    assert.NonNil(err)
    assert.Equal(t, msg, "")
}
```

The Golang FAQ does cover the topic "[Why does Go not have assertions?](https://go.dev/doc/faq#assertions)" but I interpret this as not using assertions within production code (non-test code) itself rather than in unit tests.