# Go mod gitlab subgroup module

- [Intro](#intro)
- [Create go mod in gitlab subgroup](#create-go-mod-in-gitlab-subgroup)
- [Test use go mod in gitlab subgroup](#test-use-go-mod-in-gitlab-subgroup)
- [Config ~/.netrc](#config-netrc)
- [Add go module in gitlab subgroup](#add-go-module-in-gitlab-subgroup)
- [Test in main.go](#test-in-maingo)

# Intro

In this article, we will learn how to use go mod to get private gitlab subgroup module.

# Create go mod in gitlab subgroup

First, let's create subgroup with name `group1` in `aoaojiaoaoaojiao` gitlab group.

Next, we will create a new project named `math` under subgroup `group1` in gitlab.

After that, we initialize the project using `go mod init gitlab.com/aoaojiaoaoaojiao/group1/math` in the `math` project.

Now, here is the `go.mod` file:

```
module gitlab.com/aoaojiaoaoaojiao/group1/math

go 1.20
```

We add a `Add` function in `add.go`:

`add.go`:

```go

package math

func Add(a int, b int) int {
	return a + b
}
```

# Test use go mod in gitlab subgroup

Create a golang project:

```bash
mkdir -p test-go-module-in-gitlab-subgroup
cd test-go-module-in-gitlab-subgroup
go mod init test
```

# Config ~/.netrc

Before you can pull dependency using `go get`, you need to add configure to `~/.netrc`:

```
machine gitlab.com login <gitlab login email address, i.e myname@gmail.com> password <gitlab private token>
```

# Add go module in gitlab subgroup

Then add `math` as dependency:

```bash
go get gitlab.com/aoaojiaoaoaojiao/group1/math
```

You expected everything goes well, but you didn't. Error occurred.

```
gitlab.com/aoaojiaoaoaojiao/group1/math@v0.0.0-20230530092926-88bf01cac6da: verifying module: gitlab.com/aoaojiaoaoaojiao/group1/math@v0.0.0-20230530092926-88bf01cac6da: reading https://goproxy.io/sumdb/sum.golang.org/lookup/gitlab.com/aoaojiaoaoaojiao/group1/math@v0.0.0-20230530092926-88bf01cac6da: 404 Not Found
        server response:
        not found: gitlab.com/aoaojiaoaoaojiao/group1/math@v0.0.0-20230530092926-88bf01cac6da: invalid version: git ls-remote -q origin in /tmp/gopath/pkg/mod/cache/vcs/b401f5b06f1a57210edcb631d77909880fab25833fcdeab7b9341e5d4617599b: exit status 128:
                fatal: could not read Username for 'https://gitlab.com': terminal prompts disabled
        Confirm the import path was entered correctly.
        If this is a private repository, see https://golang.org/doc/faq#git_https for additional information.
```

You need to tell to by using `export GOPRIVATE='gitlab.com'` or `go env -w GOPRIVATE=gitlab.com`:

```bash
export GOPRIVATE='gitlab.com'
go get gitlab.com/aoaojiaoaoaojiao/group1/math
```

> Why this happens is that go get tries to discover the modules at a given path in order to find the requested Go module repository. Only after the repository is found, the tools will do git clone or git checkout and the SSH keys will be used for authentication. The issue comes down to the fact that private Gitlab subgroups cannot be listed/viewed without a Gitlab Access Token.

Output:

```
go: added gitlab.com/aoaojiaoaoaojiao/group1/math v0.0.0-20230530092926-88bf01cac6da
```

# Test in main.go

Write code in `main.go` to call `Add` function in `math` module.

`main.go`

```go
package main

import (
	"fmt"

	"gitlab.com/aoaojiaoaoaojiao/group1/math"
)

func main() {
	res := math.Add(1, 2)
	fmt.Printf("1 + 2 = %d\n", res)
}
```

Output:

```bash
1 + 2 = 3
```
