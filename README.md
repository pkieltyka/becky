# Becky -- Go asset embedding for use with `go generate`

Becky embeds assets as string literals in Go source.


## `go generate`

Just copy
[`asset.go`](https://raw.githubusercontent.com/tv42/becky/master/asset.go)
in your source tree, and in your Go source do

``` go
//go:generate -command asset go run asset.go
//go:generate asset index.html
```

and run

``` console
$ go generate
```

This will create new files, named `*.gen.go`. You should add those
into your version control system, to ensure `go get` works for others.

You can pass multiple asset files at once, or repeat the `go:generate`
line.


## Variable name

The generated files declare a variable that now contains your asset.
Given the above `index.html`, the variable will be named `index`.

You can override the name with `-var=NAME`, or skip it with `-var=_`
and use side effects in your wrapper function (discussed later).

The asset will be an value of `type asset` (this code is
autogenerated, you don't need to type it in):

``` go
type asset struct {
	Name    string
	Content string
	...
}
```

Name has the original filename, as a hint for `Content-Type`
selection.


## Wrapper

For most uses, an `asset` value needs to be given application or file
type specific functionality. To make this easy, the asset value will
be passed to a function caller *wrapper*, default wrapper being the
(final) extension of the filename. For `index.html`, that's `html`.

You can override the wrapper with `-wrap=NAME`.

In your application, you'd do something like

``` go
func html(a asset) http.Handler {
	return a
}
```

or

``` go
func txt(a asset) string {
	return a.Content
}
```

or

``` go
func tmpl(a asset) *template.Template {
	return template.Must(template.New(a.Name).Parse(a.Content))
}
```

to smartly handle `*.html`, `*.txt` and `*.tmpl` assets. Feel free
to pass the fields of `asset` to a factory function or type that
matches what you need, or use the `asset`, whatever suits your
project.


## HTTP

Type `asset` implements `http.Handler`, including `ETag` cache
validation. It uses `http.ServeContent` which will set `Content-Type`
from the file name or content, and handle `Range` requests.


## Build speed

`gc`, the Go compiler, can slow down with large source files. As e.g.
image assets can get big, this can start to slow down your builds. The
mechanism used for embedding has been chosen to be the most efficient
available.

Embedding a 10MB asset (creating a 28MB Go source file) takes <1
second to generate the code and about 1 second for every compilation.

You can minimize the number of times assets need to be compiled by
putting them in a different package that updates less often than most
of your source.


## Development mode

If you build your application with `-tags dev`, `asset.ServeHTTP` will
reload the asset from disk on every request, and not use the bundled
copy. This makes editing HTML, CSS and such more convenient.


## Versioning

By including `asset.go` in your source tree, you're isolating yourself
from changes to the upstream project. Given the input-driven nature of
asset embedding, if it works once, it'll probably keep working for
you.

Feel free to grab a newer `asset.go` every now and then, or not; if a
new version for some reason doesn't work, just don't commit it to your
repo and keep using the old version. Bug reports are welcome.

If you want, you can also just install the executable with `go get
github.com/tv42/becky`, and use it like `becky index.html`.


## Go source as asset

If you are embedding Go source files as assets, and are using `go run
asset.go`, note that `go run` needs to be told where source files to
run end and where arguments start. Example:

``` go
//go:generate -command asset go run asset.go
//go:generate asset -- example.go
```
