# GRPC golang output path

- [go code output path](#go-code-output-path)
  - [code structure](#code-structure)
  - [--go_out=paths=import](#--go_outpathsimport)
  - [--go_out=module=$PREFIX](#--go_outmoduleprefix)
  - [--go_out=paths=source_relative](#--go_outpathssource_relative)

# go code output path

The protocol buffer compiler produces Go output when invoked with the go_out flag. The argument to the go_out flag is the directory where you want the compiler to write your Go output. The compiler creates a single source file for each `.proto` file input. The name of the output file is created by replacing the `.proto` extension with `.pb.go`.

Where in the output directory the generated `.pb.go` file is placed depends on the compiler flags. There are several output modes:

- If the `paths=import` flag is specified, the output file is placed in a directory named after the Go package’s import path. For example, an input file protos/buzz.proto with a Go import path of `example.com/project/protos/fizz` results in an output file at `example.com/project/protos/fizz/buzz.pb.go`. This is the default output mode if a paths flag is not specified.
- If the `module=$PREFIX` flag is specified, the output file is placed in a directory named after the Go package’s import path, but with the specified directory prefix removed from the output filename. For example, an input file `protos/buzz.proto` with a Go import path of `example.com/project/protos/fizz` and `example.com/project` specified as the module prefix results in an output file at `protos/fizz/buzz.pb.go`. Generating any Go packages outside the module path results in an error. This mode is useful for outputting generated files directly into a Go module.
- If the `paths=source_relative` flag is specified, the output file is placed in the same relative directory as the input file. For example, an input file `protos/buzz.proto` results in an output file at `protos/buzz.pb.go`

## code structure

The directory tree looks like this:

```bash
❯ exa -l --tree
drwxr-xr-x    - dylan 15 Feb 10:49 .
drwxr-xr-x    - dylan 15 Feb 10:27 ├── cmd
drwxr-xr-x    - dylan 15 Feb 10:28 │  └── greeting-server
.rw-r--r--   30 dylan 15 Feb 10:28 │     └── main.go
.rw-r--r--  250 dylan 15 Feb 10:49 ├── Makefile
drwxr-xr-x    - dylan 15 Feb 10:49 ├── protos
.rw-r--r--  365 dylan 15 Feb 10:48 │  └── greeting.proto
.rw-r--r-- 1.4k dylan 15 Feb 10:40 └── README.md
```

## --go_out=paths=import

Using `--go_out=paths=import` to generate code into import path.

```bash
❯ protoc --go_out=. --go_opt=paths=import --go-grpc_out=. --go-grpc_opt=paths=import protos/*.proto
```

```bash
❯ exa -l --tree
drwxr-xr-x    - dylan 15 Feb 11:21 .
drwxr-xr-x    - dylan 15 Feb 10:27 ├── cmd
drwxr-xr-x    - dylan 15 Feb 10:28 │  └── greeting-server
.rw-r--r--   30 dylan 15 Feb 10:28 │     └── main.go
drwxr-xr-x    - dylan 15 Feb 11:21 ├── github.com
drwxr-xr-x    - dylan 15 Feb 11:21 │  └── grpc-greeting
drwxr-xr-x    - dylan 15 Feb 11:21 │     └── greeting
.rw-r--r-- 7.2k dylan 15 Feb 11:21 │        ├── greeting.pb.go
.rw-r--r-- 3.7k dylan 15 Feb 11:21 │        └── greeting_grpc.pb.go
.rw-r--r--  250 dylan 15 Feb 10:49 ├── Makefile
drwxr-xr-x    - dylan 15 Feb 11:20 ├── protos
.rw-r--r--  365 dylan 15 Feb 10:48 │  └── greeting.proto
.rw-r--r-- 1.4k dylan 15 Feb 10:40 └── README.md
```

You can write generated code into different places. Here we put in `whatever` folder.

```bash
mkdir -p whatever
protoc --go_out=whatever --go_opt=paths=import --go-grpc_out=. --go-grpc_opt=paths=import protos/*.proto
```

The directory tree looks like this:

```bash
drwxr-xr-x    - dylan 15 Feb 11:22 .
drwxr-xr-x    - dylan 15 Feb 10:27 ├── cmd
drwxr-xr-x    - dylan 15 Feb 10:28 │  └── greeting-server
.rw-r--r--   30 dylan 15 Feb 10:28 │     └── main.go
drwxr-xr-x    - dylan 15 Feb 11:21 ├── github.com
drwxr-xr-x    - dylan 15 Feb 11:21 │  └── grpc-greeting
drwxr-xr-x    - dylan 15 Feb 11:21 │     └── greeting
.rw-r--r-- 7.2k dylan 15 Feb 11:21 │        ├── greeting.pb.go
.rw-r--r-- 3.7k dylan 15 Feb 11:22 │        └── greeting_grpc.pb.go
.rw-r--r--  250 dylan 15 Feb 10:49 ├── Makefile
drwxr-xr-x    - dylan 15 Feb 11:20 ├── protos
.rw-r--r--  365 dylan 15 Feb 10:48 │  └── greeting.proto
.rw-r--r-- 1.4k dylan 15 Feb 10:40 ├── README.md
drwxr-xr-x    - dylan 15 Feb 11:22 └── whatever
drwxr-xr-x    - dylan 15 Feb 11:22    └── github.com
drwxr-xr-x    - dylan 15 Feb 11:22       └── grpc-greeting
drwxr-xr-x    - dylan 15 Feb 11:22          └── greeting
.rw-r--r-- 7.2k dylan 15 Feb 11:22             └── greeting.pb.go
```

## --go_out=module=$PREFIX

We can put generated go code and grpc code into path, with specific directory prefix removed. Here we remove `github.com` directory and use `grpc-greeting` as the root path.

```bash
protoc --go_out=. --go_opt=module=github.com --go-grpc_out=. --go-grpc_opt=paths=import protos/*.proto
```

The directory tree looks like this:

```
drwxr-xr-x    - dylan 15 Feb 11:31 .
drwxr-xr-x    - dylan 15 Feb 10:27 ├── cmd
drwxr-xr-x    - dylan 15 Feb 10:28 │  └── greeting-server
.rw-r--r--   30 dylan 15 Feb 10:28 │     └── main.go
drwxr-xr-x    - dylan 15 Feb 11:31 ├── grpc-greeting
drwxr-xr-x    - dylan 15 Feb 11:31 │  └── greeting
.rw-r--r-- 7.2k dylan 15 Feb 11:31 │     ├── greeting.pb.go
.rw-r--r-- 3.7k dylan 15 Feb 11:31 │     └── greeting_grpc.pb.go
.rw-r--r--  250 dylan 15 Feb 10:49 ├── Makefile
drwxr-xr-x    - dylan 15 Feb 11:20 ├── protos
.rw-r--r--  365 dylan 15 Feb 10:48 │  └── greeting.proto
.rw-r--r-- 1.4k dylan 15 Feb 10:40 └── README.md
```

We can also put generated go code and grpc code into different path.

```bash
protoc --go_out=. --go_opt=module=github.com --go-grpc_out=. --go-grpc_opt=paths=import protos/*.proto
```

The directory tree looks like this:

```bash
❯ exa -l --tree
drwxr-xr-x    - dylan 15 Feb 11:27 .
drwxr-xr-x    - dylan 15 Feb 10:27 ├── cmd
drwxr-xr-x    - dylan 15 Feb 10:28 │  └── greeting-server
.rw-r--r--   30 dylan 15 Feb 10:28 │     └── main.go
drwxr-xr-x    - dylan 15 Feb 11:27 ├── github.com
drwxr-xr-x    - dylan 15 Feb 11:27 │  └── grpc-greeting
drwxr-xr-x    - dylan 15 Feb 11:27 │     └── greeting
.rw-r--r-- 3.7k dylan 15 Feb 11:27 │        └── greeting_grpc.pb.go
drwxr-xr-x    - dylan 15 Feb 11:27 ├── grpc-greeting
drwxr-xr-x    - dylan 15 Feb 11:27 │  └── greeting
.rw-r--r-- 7.2k dylan 15 Feb 11:27 │     └── greeting.pb.go
.rw-r--r--  250 dylan 15 Feb 10:49 ├── Makefile
drwxr-xr-x    - dylan 15 Feb 11:20 ├── protos
.rw-r--r--  365 dylan 15 Feb 10:48 │  └── greeting.proto
.rw-r--r-- 1.4k dylan 15 Feb 10:40 └── README.md
```

## --go_out=paths=source_relative

Using `--go_out=paths=source_relative` to generate code into the same relative directory as the input path.

```bash
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative protos/*.proto
```

The directory tree looks like this:

```bash
❯ exa -l --tree
drwxr-xr-x    - dylan 15 Feb 10:49 .
drwxr-xr-x    - dylan 15 Feb 10:27 ├── cmd
drwxr-xr-x    - dylan 15 Feb 10:28 │  └── greeting-server
.rw-r--r--   30 dylan 15 Feb 10:28 │     └── main.go
.rw-r--r--  250 dylan 15 Feb 10:49 ├── Makefile
drwxr-xr-x    - dylan 15 Feb 11:19 ├── protos
.rw-r--r-- 7.2k dylan 15 Feb 11:19 │  ├── greeting.pb.go
.rw-r--r--  365 dylan 15 Feb 10:48 │  ├── greeting.proto
.rw-r--r-- 3.7k dylan 15 Feb 11:19 │  └── greeting_grpc.pb.go
.rw-r--r-- 1.4k dylan 15 Feb 10:40 └── README.md
```
