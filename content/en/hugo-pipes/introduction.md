---
title: Hugo Pipes Introduction
linkTitle: Hugo Pipes
description: Hugo Pipes is Hugo's asset processing set of functions.
date: 2018-07-14
publishdate: 2018-07-14
categories: [asset management]
keywords: []
menu:
  docs:
    parent: "pipes"
    weight: 20
weight: 01
sections_weight: 01
draft: false
toc: true
aliases: [/assets/]
---

## Find Resources in /assets

This is about the global Resources mounted inside `/assets`. For the `.Page` scoped Resources, see [Page Resources](/content-management/page-resources/).

Note that you can mount any directory into Hugo's virtual `assets` folder using the [Mount Configuration](/hugo-modules/configuration/#module-config-mounts).

| Function  | Description |
| ------------- | ------------- |
| `resources.Get`  | Get locates the filename given in Hugo's assets filesystem and creates a `Resource` object that can be used for further transformations. See [Get Resource with resources.Get and resources.GetRemote](#get-resource-with-resourcesget-and-resourcesgetremote).  |
| `resources.GetRemote`  | Same as `Get`, but it accepts remote URLs. See [Get Resource with resources.Get and resources.GetRemote](#get-resource-with-resourcesget-and-resourcesgetremote).|
| `resources.GetMatch`  | `GetMatch` finds the first Resource matching the given pattern, or nil if none found. See Match for a more complete explanation about the rules used. |
| `resources.Match`  | `Match` gets all resources matching the given base path prefix, e.g "*.png" will match all png files. The "*" does not match path delimiters (/), so if you organize your resources in sub-folders, you need to be explicit about it, e.g.: "images/*.png". To match any PNG image anywhere in the bundle you can do "\*\*.png", and to match all PNG images below the images folder, use "images/\*\*.jpg". The matching is case insensitive. Match matches by using the files name with path relative to the file system root with Unix style slashes (/) and no leading slash, e.g. "images/logo.png". See https://github.com/gobwas/glob for the full rules set.|

See the [GoDoc Page](https://pkg.go.dev/github.com/gohugoio/hugo@v0.93.1/tpl/resources) for the `resources` package for an up to date overview of all template functions in this namespace.

## Get Resource with resources.Get and resources.GetRemote

In order to process an asset with Hugo Pipes, it must be retrieved as a `Resource` using `resources.Get` or `resources.GetRemote`.

With `resources.Get`, the first argument is a local path relative to the `assets` directory/directories:

```go-html-template
{{ $local := resources.Get "sass/main.scss" }}
```

With `resources.GetRemote`, the first argument is a remote URL:

```go-html-template
{{ $remote := resources.GetRemote "https://www.example.com/styles.scss" }}
```

`resources.Get` and `resources.GetRemote` return `nil` if the resource is not found.

## Copy a Resource

{{< new-in "0.100.0" >}}

`resources.Copy` allows you to copy almost any Hugo `Resource` (the one exception is the `Page`), possibly most useful for renaming things:

```go-html-template
{{ $resized := $image.Resize "400x400" |  resources.Copy "images/mynewname.jpg" }}
<img src="{{ $resized.RelPermalink }}">
```

### Caching

By default, Hugo calculates a cache key based on the `URL` and the `options` (e.g. headers) given.

{{< new-in "0.97.0" >}} You can override this by setting a `key` in the options map. This can be used to get more fine grained control over how often a remote resource is fetched, e.g.:


```go-html-template
{{ $cacheKey := print $url (now.Format "2006-01-02") }}
{{ $resource := resource.GetRemote $url (dict "key" $cacheKey) }}
```

### Error Handling

{{< new-in "0.91.0" >}}

The return value from `resources.GetRemote` includes an `.Err` method that will return an error if the call failed. If you want to just log any error as a `WARNING` you can use a construct similar to the one below.

```go-html-template
{{ with resources.GetRemote "https://gohugo.io/images/gohugoio-card-1.png" }}
  {{ with .Err }}
    {{ warnf "%s" . }}
  {{ else }}
    <img src="{{ .RelPermalink }}" width="{{ .Width }}" height="{{ .Height }}" alt="">
  {{ end }}
{{ end }}
```

Note that if you do not handle `.Err` yourself, Hugo will fail the build the first time you start using the `Resource` object.

### Remote Options

When fetching a remote `Resource`, `resources.GetRemote` takes an optional options map as the last argument, e.g.:

```go-html-template
{{ $resource := resources.GetRemote "https://example.org/api" (dict "headers" (dict "Authorization" "Bearer abcd"))  }}
```

If you need multiple values for the same header key, use a slice:

```go-html-template
{{ $resource := resources.GetRemote "https://example.org/api"  (dict "headers" (dict "X-List" (slice "a" "b" "c")))  }}
```

You can also change the request method and set the request body:

```go-html-template
{{ $postResponse := resources.GetRemote "https://example.org/api"  (dict 
    "method" "post"
    "body" `{"complete": true}` 
    "headers" (dict 
        "Content-Type" "application/json"
    )
)}}
```

### Caching of Remote Resources

Remote resources fetched with `resources.GetRemote` will be cached on disk. See [Configure File Caches](/getting-started/configuration/#configure-file-caches) for details.

## Asset directory

Asset files must be stored in the asset directory. This is `/assets` by default, but can be configured via the configuration file's `assetDir` key.

### Asset Publishing

Hugo publishes assets to the to the `publishDir` (typically `public`) when you invoke `.Permalink`, `.RelPermalink`, or `.Publish`. You can use `.Content` to inline the asset.

## Go Pipes

For improved readability, the Hugo Pipes examples of this documentation will be written using [Go Pipes](/templates/introduction/#pipes):

```go-html-template
{{ $style := resources.Get "sass/main.scss" | resources.ToCSS | resources.Minify | resources.Fingerprint }}
<link rel="stylesheet" href="{{ $style.Permalink }}">
```

## Method aliases

Each Hugo Pipes `resources` transformation method uses a __camelCased__ alias (`toCSS` for `resources.ToCSS`).
Non-transformation methods deprived of such aliases are `resources.Get`, `resources.FromString`, `resources.ExecuteAsTemplate` and `resources.Concat`.

The example above can therefore also be written as follows:

```go-html-template
{{ $style := resources.Get "sass/main.scss" | toCSS | minify | fingerprint }}
<link rel="stylesheet" href="{{ $style.Permalink }}">
```

## Caching

Hugo Pipes invocations are cached based on the entire *pipe chain*.

An example of a pipe chain is:

```go-html-template
{{ $mainJs := resources.Get "js/main.js" | js.Build "main.js" | minify | fingerprint }}
```

The pipe chain is only invoked the first time it is encountered in a site build, and results are otherwise loaded from cache. As such, Hugo Pipes can be used in templates which are executed thousands or millions of times without negatively impacting the build performance.
