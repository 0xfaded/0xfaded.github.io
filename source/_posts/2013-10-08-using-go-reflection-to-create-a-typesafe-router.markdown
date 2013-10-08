---
layout: post
title: "Using Go reflection to create a typesafe router"
date: 2013-10-08 00:30
comments: true
categories: [go, reflection, http]
---

Foreword
--------
If you are simply looking for a regular expression based router to work with
the Go `net/http` package, [Gorilla Mux](http://www.gorillatoolkit.org/pkg/mux)
is a mature option. This post is to demonstrate a simple use case for
reflection in Go.

Motivation
----------
One feature of many web frameworks lacking from Go's `net/http` package is URL
routing based on regular expressions. As an example, user code might listen for
requests to `/user/(id=[0-9]+)/` and expect to be invoked when paths like
`/user/1` and `/user/42` are requested.

[Several](https://github.com/gorilla/mux)
[other](https://github.com/stretchr/goweb)
implementations already exist that extract a `map[string]string`
containing the (named) dynamic parts of the URL. This map is then passed
to user code which handles the request. Using reflection, we can go
one better and provide the user code with a pre-populated struct. As an
added bonus, the routes can be made type safe based on the struct fields.

Registering Routes
------------------
As done in the `net/http` package, handlers listen to routes defined by a pattern.
In addition, the handler will also need to register the format it expects to
receive the dynamic components of the URL, so called URL parameters. I have
chosen to have the registering code provide a zero value struct that the router
inspects via reflection.

``` go
    type DoParams struct {
        UserId int64 `rhttp:"id"`
    }
    func Do(w http.ResponseWriter, r *http.Request, i interface{}) {
        ...
    }
    func main() {
        router := rhttp.NewRouter()
        router.HandleFunc("/user/(id=.*)", Do, &DoParams{})
        ...
    }
```

Note the use of the `` `rhttp:"id"` `` struct tag which is used to name the
the parameter.

Inspecting the Params
---------------------
Go makes reflection a straight forward process. The first step is to validate
that `params` is indeed a pointer to a struct (this is merely a requirement
of my implementation). There is also the possibility of a nil interface, or
an interface containing a nil pointer. The two possibilities are distinct,
similar to the way an empty slice and a nil slice differ. I won't go into
the details.
``` go
    func readParams(params {}interface) (...) {
        v := reflect.ValueOf(params)
        if params == nil || v.Kind() == reflect.Ptr && v.IsNil() {
            return
        }
        ...
```
This snippet introduces `Value.Kind()`, which may seem odd as a pointer is
usually referred to as a `Type`. In type theory, a type is simply a set of
values, and a kind is a set of types. Types in Go are user defined, one can
define `T1` and `T2` as follows:
``` go
    type T1 int
    type T2 int
```
Without getting bogged down in theory, `T1` and `T2` are both of kind `int`. We
don't care what type of pointer we are inspecting, we do care that it is a
pointer.

Assuming our pointer is non-nil, accessing the underlying struct is now straight
forward.

``` go
    ...
    if v.Kind() == reflect.Ptr {
        v = v.Elem() // dereference the pointer
    } else {
        err = errors.New(fmt.Sprintf("params not a struct pointer (%s)", v.Kind()))
        return
    }
    if v.Kind() != reflect.Struct {
        err = errors.New(fmt.Sprintf("*params is not a struct (%s)", v.Kind()))
        return
    }
    ...
```

Traversing over the struct is also pleasantly simple. Parameters are named
either explicitly by a rhttp tag, or in its absence the field name.

``` go
    ...
    paramsType = v.Type()
    for i := 0; i < paramsType.NumField(); i += 1 {
        field := paramsType.Field(i)
        if tag := field.Tag.Get("rhttp"); len(tag) != 0 {
            // Add a parameter to scope with name `tag'
        } else if isValidName(field.Name) {
            // Add a parameter to scope with name `field.Name'
        }
    }
    ...
```

Building Params
---------------
For thread safety, a params struct is allocated per invocation. The URL is
parsed for its dynamic components, and each is assigned to a designated struct
field. The designation here has been precomputed as `paramsIndex`, and the
slice of string parameter values is `vals`. An error is returned if a field's
type prevents a value from being parsed.

``` go
    params := reflect.New(paramsType)
    for f, v := range paramsIndex {
        value := params.Elem().Field(f)
        switch value.Type().Kind() {
        case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
            if val, err := strconv.ParseInt(vals[v], 10, value.Type().Bits()); err != nil {
                return nil, err
            } else {
                value.SetInt(val)
            }
        case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
            ...
        }
    }
    return params.Interface(), nil
```
One useful trick for when marshalling strings into data is the use of `Type.Bits()`
to determine the bit width of the underlying type. This may be enough to save from
handling each width separately, as above. The final step is to create a
`interface{}` with `Value.Interface()` usable by regular code.

Recovering Params in User Code
------------------------------
Unfortunately, Go's type system is not powerful enough to pass the struct pointer
directly back to user code, and must pass an `interface{}` instead. Even Haskell
would require its [GADT's](http://en.wikibooks.org/wiki/Haskell/GADT) extension
to achieve this. Therefore, the user is going to have to make a runtime type
assertion and convert the interface back to the registered struct type. A
complete example below.

``` go
    type Id int64
    type DoParams struct {
        UserId Id `rhttp:"id"`
        Action string `rhttp:"targ"`
        ...
    }
    func Do(w http.ResponseWriter, r *http.Request, i interface{}) {
        params := i.(*DoParams)
        ...
    }
    func main() {
        router := rhttp.NewRouter()
        router.HandleFunc("/user/(id=.*)/action/(targ=[abc])", Do, &DoParams{})
        http.Handle("/", router)
    }
```

Conclusion
----------
Go makes reflection a first class usage pattern, which my limited experience
tells me is capable of replicating functionality of languages with more
sophisticated type systems such as Haskell or C++. The downside is that
reflection is available only at runtime, unlike templates or other type system
sorcery seen in various functional languages. The upside is the simplicity and
ease of use which aligns with Go's philosophy of making small performance
sacrifices for usability gains.

The code for the router is available on [github](https://github.com/0xfaded/rhttp)
and obviously includes substantially more machinery than has been shown. The
only true advantage over existing solutions is the automatic unmarshalling of
parameters, although this does allow neat tricks like overloading patterns with
two handlers accepting differently typed parameters. The resolution process
uses binary search to match the non-dynamic parts of the expression, which
may be of interest if you have thousands of routes defined. Other packages that
I've looked at, including the built-in `net/http`, check every registered pattern.

I still recommend using a more mainstream solution though.
