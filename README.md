# A Templating Library in Plain XQuery

eXist's HTML templating is only usable for HTML, but for app generation tasks we needed a library, which is able to process both, XML/HTML and plain text files. The module was inspired by other templating languages like [nunjucks](https://mozilla.github.io/nunjucks/) or *jsx*, but uses the full power of XPath for expressions. It was also designed to be backwards-compatible with the simpler templating syntax TEI Publisher uses within ODD, further extending the possibilities available within the `pb:template` element in ODD.

Instead of being entirely based on regular expressions, the templating module implements a parser generating an abstract syntax tree (AST) in XML. The AST is then compiled into XQuery code, which - when executed - produces the final output.

## Expressions

The template syntax is similar to [nunjucks](https://mozilla.github.io/nunjucks/) or [jinja](https://jinja.palletsprojects.com/en/3.1.x/templates/), but uses the host language for all expressions, therefore giving users the full power of XPath/XQuery.

The templating is passed a context map, which should containing all the information necessary for processing the template expressions. Each top-level property in the context map is made available as an XQuery variable. So if you have a context map like

```xquery
map {
    "title": "my title",
    "theme": map {
        "fonts": map {
            "content": "serif"
        }
    }
}
```

you can use a value expression `[[$title]]` to output the title. And to insert the content font, use `[[$theme?fonts?content]]`.

Supported template expressions are:

| Expression | Description |
| -------- | ------- |
| `[[ expr ]]` | Insert result of evaluating `expr` |
| `[% if expr %] … [% endif %]` | Conditional evaluation of block |
| `… [% elsif expr %] …` | *else if* block after *if* |
| `… [% else %] … [% endif %]` | *else* block after *if* or *else if* |
| `[% for $var in expr %] … [% endfor %]` | Loop `$var` over sequence returned by `expr` |
| `[% include expr %]` | Include a partial. `expr` should resolve to relative path. |
| `[% extends expr %]` | Extend a base template: contents of child template passed to base template in variable `$content`. Named blocks in child overwrite blocks in base. |
| `[% block name %] … [% endblock %]` | Defines a named block or overwrites corresponding block in base template. |
| `[% import "uri" as "prefix" at "path" %]` | Import an XQuery module so its functions/variables can be used in template expressions. |
| `[# … #]` | Single or multi-line comment: content will be discarded |

`expr` must be a valid XPath expression.

For some real pages built with jinks-templates, check the main [jinks app manager](https://github.com/eeditiones/jinks/tree/main/pages). This app also includes a playground and demo for jinks-templates.

## Output Modes

The library supports two modes: **XML/HTML** and **plain text**. They differ in the XQuery code templates are compiled into. While the first will always return XML – and fails if the result is not well-formed, the second uses XQuery string templates.

## Use in XQuery

The library exposes one main function, `tmpl:process`, which takes 3 arguments:

1. the template to process as a string
2. the context providing the information to be passed to templating expressions
3. a configuration map with the following properties:
   1. `plainText`: should be true for plain text processing (default is false)
   2. `resolver`: the resolver function to use (see below)
   3. `modules`: sequence of modules to import (see below)
   4. `debug`: if true, `tmpl:process` returns a map with the result, ast and generated XQuery code (default is false)

A simple example:

```xquery
xquery version "3.1";

import module namespace tmpl="http://e-editiones.org/xquery/templates";

let $input :=
    <body>
        <h1>[[$title]]</h1>
        <p>You are running eXist [[system:get-version()]]</p>
    </body>
    => serialize()
let $context := map {
    "title": "My app"
}
return
    tmpl:process($input, $context, map { "plainText": false() })
```

The input is constructed as XML, but serialized into a string for the call to `tmpl:process`. The context map contains a single property, which will become available as variable `$title` within template expressions.

### Specifying a resolver

The `resolver` function is needed if you would like to use `[% include %]`, `[% extends %]` or `[% import %]` in your templates. It should point to a function with one parameter: the relative path to the resource, and should return a map with two fields:

* `path`: the absolute path to the resource
* `content`: the content of the resource as a string

If the resource cannot be resolved, the empty sequence should be returned. In the following example we're prepending the assumed application root (`$config:app-root`) to get an absolute path and load the resource:

```xquery
import module namespace tmpl="http://e-editiones.org/xquery/templates";
import module namespace config=...;

declare function local:resolver($relPath as xs:string) as map(*)? {
    let $path := $config:app-root || "/" || $relPath
    let $content :=
        if (util:binary-doc-available($path)) then
            util:binary-doc($path) => util:binary-to-string()
        else if (doc-available($path)) then
            doc($path) => serialize()
        else
            ()
    return
        if ($content) then
            map {
                "path": $path,
                "content": $content
            }
        else
            ()
};

let $input :=
    <body>
        <h1>[[$title]]</h1>
        <p>You are running eXist [[system:get-version()]]</p>
    </body>
    => serialize()
let $context := map {
    "title": "My app"
}
let $config := map {
    "resolver": local:resolver#1
}
return
    tmpl:process($input, $context, $config)
```

### Importing XQuery modules

To make the variables and functions of specific XQuery modules available in your templates, you have to explicitely list those in the configuration using property `modules`. This is a list of maps, each defining one module to import:

```xquery
let $config := map {
    "resolver": local:resolver#1,
    "modules": map {
        "uri": "http://www.tei-c.org/tei-simple/config",
        "prefix": "config",
        "at": $config:app-root || "/modules/config.xqm"
    }
}
return
    tmpl:process($input, $context, $config)
```

## Use frontmatter to extend the context

Templates may start with a frontmatter block enclosed in `---`. The purpose of the frontmatter is to extend or overwrite the static context map provided in the second argument to `tmpl:process`. Currently only JSON syntax is supported. The frontmatter block will be parsed into an JSON object and merged with the static context passed to `tmpl:process`. For example, take the following template:

```html
---json
{
  "title": "Lorem ipsum dolor sit amet",
  "author": "Hans"
}
---
<article>
<h1>[[ $title ]]</h1>

<p>Consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.</p>

<footer>Published [[format-date(current-dateTime(), "[MNn] [D], [Y]", "en", (), ())]] by [[$author]].</footer>
</article>
```

This will overwrite the `title` and `author` properties of the static context map. The frontmatter block should come first in the file with a newline after each of the two separators. However, to allow for well-formed XML, the frontmatter may come *after* one or more surrounding elements, e.g.:

```html
<article>
---json
{
  "title": "Lorem ipsum dolor sit amet",
  "author": "Hans"
}
---
<h1>[[ $title ]]</h1>
</article>
```