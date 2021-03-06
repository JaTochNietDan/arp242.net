---
layout: post
title: "Using flags for configuration in Go"
tags: ['Go', 'Unix', 'Config']
---

I like commandline flags for configuration. It's simple, robust, flexible, and
generally just bullshit-free.
See [*flags are great for configuration*](/flags-config.html).

The way I do it in Go is keeping a list of exported variables in a `cfg`
package:

    // Package cfg handles the application configuration.
    package cfg

    // Configuration variables.
    var (
        Prod         bool   // Production mode; hide errors.
        Domain       string // Domain, including protocol.
        // [..]
    )

    // Set configuration variables from os.Args.
    func Set() {
        flag.BoolVar(&Prod, "prod", false, "Production mode; hide errors.")
        flag.StringVar(&Domain, "domain", "http://localhost:8080", "Domain, including protocol.")
        // [..]
    }

And then call `cfg.Set()` from your `main()` function.

That's it. Somewhere north of 75% of all applications will never need anything
more than this.

I often use a simple program to generate the `cfg.go` file as it avoids
duplicating the flag and variable names a few times. See the end of this post.

If all of this seems really trivial to you then you're correct. Yet I encounter
plenty of fairly simple projects (Go and others) in the wild that have complex
configuration schemes just to load a few settings.

Using flags for large and complex applications is not a good idea. Don't try it
for your Postfix competitor. But most applications are not complex. The project
I worked on this week has 2.8k lines of code and 8 settings. You don't need 2.2k
lines of code from Viper (and 9.5k from YAML lib) for that. It doesn't really
make things [easier][easy] for anyone.


---

Some people might object to the use of package globals. I think it's fine. It's
a tiny package (no room for confusion), they're set only once (no race
conditions), there is never any reason to create more than one instance, and it
should be easy enough to set up for tests. In practice, only a few values tend
to be referenced outside of the `main()` function (usually `cfg.Prod` and
`cfg.Domain`).

I certainly consider this vastly superior to [Viper's][viper] untyped
`viper.GetBool("prod")`.

As mentioned in my other article, passing values from the environment can be
done from the commandline `prog -domain "$DOMAIN"`).

The stdlib [flag][flag] package is enough for most programs. Some other
libraries (e.g. [cobra][cobra]) make some operations a bit easier if you have a
lot of subcommands like git. The basic idea remains the same.

P.S. If you're going to use a configuration package then I recommend
[sconfig][sconfig], because that's what I wrote and it's perfect ;-)

[viper]: https://github.com/spf13/viper
[flag]: https://golang.org/pkg/flag/
[cobra]: https://github.com/spf13/cobra
[sconfig]: https://github.com/arp242/sconfig
[easy]: /easy.html


Generate script
---------------

I usually save it as `cfg/gen.go`.

Note: use `go run cfg/gen.go > cfg/cfg.go` the first time since the
`//go:generate` is in the generated file. After that `go generate ./cfg` will
work.

{% raw %}
    // +build go_run_only

    package main

    import (
        "bytes"
        "fmt"
        "go/format"
        "os"
        "strings"
        "text/template"
    )

    // Add your flags here.
    var flags = []flag{
        flag{"bool", "Prod", "prod", false, "Production mode; hide errors."},
        flag{"string", "Domain", "domain", "http://localhost:8080", "Domain, including protocol."},
    }

    type flag struct {
        Type    string      // Go type name (e.g. bool, string, etc.)
        Name    string      // Variable name in the cfg package (e.g. Listen).
        Flag    string      // Flag name (e.g. "listen")
        Default interface{} // Default value.
        Help    string      // Help text.
    }

    func main() {
        longest := 0
        for _, f := range flags {
            if len(f.Name) > longest {
                longest = len(f.Name)
            }
        }

        var buf bytes.Buffer
        err := tpl.Execute(&buf, struct {
            Flags   []flag
            Longest int
        }{flags, longest})
        if err != nil {
            _, _ = fmt.Fprintf(os.Stderr, "template: %s\n", err)
            os.Exit(1)
        }

        src, err := format.Source(buf.Bytes())
        if err != nil {
            _, _ = fmt.Fprintf(os.Stderr, "gofmt: %s\n", err)
            os.Exit(1)
        }

        fmt.Print(string(src))
    }

    var tpl = template.Must(template.New("").
        Option("missingkey=error").
        Funcs(template.FuncMap{
            "ucfirst": func(s string) string { return strings.Title(s) },
            "pad":     func(s string, l int) string { return s + strings.Repeat(" ", l-len(s)) },
        }).
        Parse(`//go:generate sh -c "go run gen.go > cfg.go"

    // Code generated by gen.go; DO NOT EDIT.

    // Package cfg handles the application configuration.
    package cfg

    import (
        "flag"
        "fmt"
    )

    // Configuration variables.
    var ({{range $f := .Flags}}
        {{$f.Name}} {{$f.Type}} // {{.Help}}{{end}}
    )

    // Set configuration variables from os.Args.
    func Set() {{"{"}}{{range $f := .Flags}}
        flag.{{$f.Type|ucfirst}}Var(&{{$f.Name}}, "{{$f.Flag}}", {{printf "%#v" $f.Default}}, "{{$f.Help}}"){{end}}
        flag.Parse()
    }

    // Print out all configuration values.
    func Print() {{"{"}}{{range $f := .Flags}}
        fmt.Printf("{{pad $f.Name $.Longest}}   %#v\n", {{$f.Name}}){{end}}
    }
    `))
{% endraw %}
