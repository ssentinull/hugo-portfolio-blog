---
title: "Create APIs using Golang | Part 1 : Workspace Setup."
date: "2021-11-16"
author: "ssentinull"
cover: "img/blogs/create-api-using-golang-setup/0.jpg"
description: "A good workspace will go a long way."
tags: ["environment", "api", "golang"]
---

## Introduction.

According to a [2021 survey](https://insights.stackoverflow.com/survey/2021#section-most-popular-technologies-programming-scripting-and-markup-languages) conducted by StackOverflow, Golang is number fourteen in terms of the most used language / tool amongst developers, right above Kotlin and below PowerShell. This is for good reasons. Its no-frills syntax, static typing, out-of-the-box build tools, and performant concurrency are a few reasons why developers, including me, prefer it over the rest. In this series of tutorials, we'll cover how to build APIs using Golang.

## Project Description.

We'll be creating a simple project that revolves around the concept of libraries. The APIs will allow us to get a catalog of books, filter books according to multiple categories, get detailed information regarding each book, as well as borrow and return them. Seems straightforward, right?

## Initialize Dependencies.

We'll use [Go Modules](https://go.dev/blog/using-go-modules) as our dependency management system and a couple of dependencies to setup our project:

- [Echo](https://echo.labstack.com/) : performant and minimalist web framework.
- [Viper](https://github.com/spf13/viper) : environment variable configurations library.
- [Logrus](https://github.com/sirupsen/logrus) : structured and pluggable logging library.

```shell
$ git init
$ go mod init github.com/your_github_username/create-apis-using-golang
$ go get github.com/labstack/echo/v4
$ go get github.com/spf13/viper
$ go get github.com/sirupsen/logrus
```

## Plan Codebase Layout.

Codebase layout might be insignificant at first glance, but trust me when I say this, it will hinder your productivity in the long run if you don't carefully plan it in the beginning. That's why I suggest following the highly rated ones instead of just creating your own from scratch, especially if you don't have a plan of how to structure your layout. In this series, we'll be using the [Go Standard Layout](https://github.com/golang-standards/project-layout), with some modifications of course. The Github page provides thorough explanations and examples behind the reasoning of the layout and based on the repo's 36k+ stars, I think it's a great place to start.

## Create Sample Web Server.

To run a sample web server, create a `main.go` inside a `/internal/cmd/server` directory. This dir is used to house our project's main application and nothing else. Our main application should only comprise mostly imports from our modules.

{{< code language="go" title="main.go" id="1" >}}

    package main

    import (
        "log"
        "net/http"
        "time"

        "github.com/labstack/echo/v4"
    )

    func main() {
        e := echo.New()
        e.GET("/", func(c echo.Context) error {
            return c.String(http.StatusOK, "Hello, World!")
        })

        s := &http.Server{
            Addr:         ":8080",
            ReadTimeout:  2 * time.Minute,
            WriteTimeout: 2 * time.Minute,
        }

        log.Fatal(e.StartServer(s))
    }

{{< /code >}}

{{< image src="/img/blogs/create-api-using-golang-setup/1.png" position="center" >}}

## Configure Logging Parameters.

To catch any errors our project might have, we need a way to effectively log our server. Thankfully, Logrus's got our back. All we need to do is set our logging parameters in our `main.go` and the configurations will be used every time we log anything in our console.

{{< code language="go" title="main.go" id="3" >}}

    package main

    import (
        "net/http"
        "os"
        "time"

        "github.com/labstack/echo/v4"
        "github.com/sirupsen/logrus"
    )

    // initialize logger configurations
    func initLogger() {
        logrus.SetFormatter(&logrus.TextFormatter{
            ForceColors:     true,
            DisableSorting:  true,
            DisableColors:   false,
            FullTimestamp:   true,
            TimestampFormat: "15:04:05 02-01-2006",
        })

        logrus.SetOutput(os.Stdout)
        logrus.SetReportCaller(true)
        logrus.SetLevel(logrus.ErrorLevel)
    }

    // run initLogger() before running main()
    func init() {
        initLogger()
    }

    func main() {
        e := echo.New()
        e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, World!")
        })

        s := &http.Server{
            Addr:         ":8080",
            ReadTimeout:  2 * time.Minute,
            WriteTimeout: 2 * time.Minute,
        }

        logrus.Fatal(e.StartServer(s))
    }

{{< /code >}}

## Use Environment Variables.

Currently, the server is running on port `8080` and it's hard-coded into the server instance. Since the port number used might change depending on the server's used ports, we need to set that value as an environment variable. Environment variables are used to store interchangeable values, account credentials, and other secrets that we don't want everyone to know.

What we need to do is create a `config.yml.example` in the root dir that will be used as a template for environment variables in case someone else clones our repo. Also, create a `.gitignore` to ignore `config.yml` files from being committed to our repo.

{{< code language="yml" title="config.yml.example" id="4" >}}

    env:
    server_port:

{{< /code >}}

{{< code language=".gitignore" title=".gitignore" id="5" >}}

    config.yml

{{< /code >}}

Copy-paste the `config.yml.example` file, rename it to `config.yml`, and set `DEV` and `8080` for `env` and `server_port` respectively. Revisiting back to the golang standard layout repo, we see that `/config` dir is stated to _store configuration file templates or default configs_. So we'll use `/internal/config` dir to place our getter functions in a file called `config.go` and call the functions in `main.go`.

{{< code language="go" title="config.go" id="6" >}}

    package config

    import (
        "strings"

        "github.com/sirupsen/logrus"
        "github.com/spf13/viper"
    )

    func GetConf() {
        viper.AddConfigPath(".")
        viper.AddConfigPath("./..")
        viper.AddConfigPath("./../..")
        viper.SetConfigName("config")
        viper.SetEnvPrefix("svc")

        replacer := strings.NewReplacer(".", "_")
        viper.SetEnvKeyReplacer(replacer)

        viper.AutomaticEnv()
        if err := viper.ReadInConfig(); err != nil {
            logrus.Warningf("%v", err)
        }
    }

    func Env() string {
        return viper.GetString("env")
    }

    func ServerPort() string {
        return viper.GetString("server_port")
    }

{{< /code >}}

{{< code language="go" title="main.go" id="7" >}}

    package main

    import (
        "net/http"
        "os"
        "time"

        "github.com/labstack/echo/v4"
        "github.com/sirupsen/logrus"
        "github.com/ssentinull/create-apis-using-golang/internal/config"
    )

    // initialize logger configurations
    func initLogger() {
        logLevel := logrus.ErrorLevel
        switch config.Env() {
        case "dev", "development":
            logLevel = logrus.InfoLevel
        }

        logrus.SetFormatter(&logrus.TextFormatter{
            ForceColors:     true,
            DisableSorting:  true,
            DisableColors:   false,
            FullTimestamp:   true,
            TimestampFormat: "15:04:05 02-01-2006",
        })

        logrus.SetOutput(os.Stdout)
        logrus.SetReportCaller(true)
        logrus.SetLevel(logLevel)
    }

    // run initLogger() before running main()
    func init() {
        initLogger()
    }

    func main() {
        e := echo.New()
        e.GET("/", func(c echo.Context) error {
            return c.String(http.StatusOK, "Hello, World!")
        })

        s := &http.Server{
            Addr:         ":" + config.ServerPort(),
            ReadTimeout:  2 * time.Minute,
            WriteTimeout: 2 * time.Minute,
        }

        logrus.Fatal(e.StartServer(s))
    }

{{< /code >}}

## Setup Daemons. :smiling_imp:

To make our life easier, we need to set up a daemon that listens to changes made in the workspace so that the server can automatically restart itself on saved changes. We do this by installing [Modd](https://github.com/cortesi/modd) and setting it to listen to files with `.go` extensions. Create a `.modd` dir in our root directory and within it a `server.modd.conf` file. Also, create a `Makefile` in the root dir to abbreviate our CLI command.

{{< code language="conf" title="server.modd.conf" id="8" >}}

    **/\*.go !**/\*\_test.go {
        daemon +sigterm: go run internal/cmd/server/main.go
    }

{{< /code >}}

{{< code language="makefile" title="Makefile" id="9" >}}

    # command to run the server in daemon mode
    run-server:
        @modd -f ./.modd/server.modd.conf

{{< /code >}}

```shell
# run the server in daemon mode
$ make run-server
```

{{< image src="/img/blogs/create-api-using-golang-setup/2.png" position="center" >}}

Our workspace is now complete!! :rocket: The next step would be the meat and potatoes of this series, which is building the logic of the APIs.

I hope this could be beneficial to you. Thank you for taking the time to read this article. :pray:

-- ssentinull
