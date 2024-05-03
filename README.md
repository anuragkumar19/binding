# binding

[![GoDoc Widget]][GoDoc]

`binding` is a go modules which assist in binding request data to struct. The source code is taken from [Echo](https://github.com/labstack/echo) and modified to support std lib and chi.

### Few things you should know

-   URL Params binding using struct tags is only supported for chi. (not std lib)
-   Echo bind both query-form and body-form in `form` tag. But in this version it only bind body when using `form` struct tag. Use `query` tag if you want to bind query params.
-   Echo doesn't bind query for method other than `GET`, `HEAD` and `DELETE` but as we not don't bind query-form in form tag, query will be binded for all methods.

## Docs

Parsing request data is a crucial part of a web application. In this package this is done with a process called _binding_. This is done with information passed by the client in the following parts of an HTTP request:

-   URL Path parameter
-   URL Query parameter
-   Header
-   Request body

This package provides different ways to perform binding, each described in the sections below.

## Struct Tag Binding

With struct binding you define a Go struct with tags specifying the data source and corresponding key. In your request handler you simply call `binding.Bind(req, i interface{})` with a pointer to your struct. The tags tell the binder everything it needs to know to load data from the request.

In this example a struct type `User` tells the binder to bind the query string parameter `id` to its string field `ID`:

```go
type User struct {
  ID string `query:"id"`
}

// in the handler for /users?id=<userID>
var user User
err := binding.Bind(req, &user); if err != nil {
    w.WriteHeader(http.StatusBadRequest)
    w.Write([]byte("bad request"))
}
```

### Data Sources

This package supports the following tags specifying data sources:

-   `query` - query parameter
-   `param` - path parameter (also called route)
-   `header` - header parameter
-   `json` - request body. Uses builtin Go [json](https://golang.org/pkg/encoding/json/) package for unmarshalling.
-   `xml` - request body. Uses builtin Go [xml](https://golang.org/pkg/encoding/xml/) package for unmarshalling.
-   `form` - form data. Values are request body only. Uses Go standard library form parsing.

### Data Types

When decoding the request body, the following data types are supported as specified by the `Content-Type` header:

-   `application/json`
-   `application/xml`
-   `application/x-www-form-urlencoded`

When binding path parameter, query parameter, header, or form data, tags must be explicitly set on each struct field. However, JSON and XML binding is done on the struct field name if the tag is omitted. This is according to the behaviour of [Go's json package](https://pkg.go.dev/encoding/json#Unmarshal).

### Multiple Sources

It is possible to specify multiple sources on the same field. In this case request data is bound in this order:

1. Path parameters
2. Query parameters
3. Request body

```go
type User struct {
  ID string `param:"id" query:"id" form:"id" json:"id" xml:"id"`
}
```

Note that binding at each stage will overwrite data bound in a previous stage. This means if your JSON request contains the query param `name=query` and body `{"name": "body"}` then the result will be `User{Name: "body"}`.

### Direct Source

It is also possible to bind data directly from a specific source:

Request body:

```go
err := binding.BindBody(req, &payload)
```

Query parameters:

```go
err := binding.BindQueryParams(req, &payload)
```

Path parameters:

```go
err := binding.BindPathParams(req, &payload)
```

Header parameters:

```go
err := binding.BindHeaders(req, &payload)
```

Note that headers is not one of the included sources with `binding.Bind`. The only way to bind header data is by calling `BindHeaders` directly.

### Security

To keep your application secure, avoid passing bound structs directly to other methods if these structs contain fields that should not be bindable. It is advisable to have a separate struct for binding and map it explicitly to your business struct.

Consider what will happen if your bound struct has an Exported field `IsAdmin bool` and the request body contains `{IsAdmin: true, Name: "hacker"}`.

### Example

In this example we define a `User` struct type with field tags to bind from `json`, `form`, or `query` request data:

```go
type User struct {
  Name  string `json:"name" form:"name" query:"name"`
  Email string `json:"email" form:"email" query:"email"`
}

type UserDTO struct {
  Name    string
  Email   string
  IsAdmin bool
}
```

And a handler at the POST `/users` route binds request data to the struct:

```go
mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
  u := new(User)
  if err := binding.Bind(u); err != nil {
    // 400- bad request
  }

  // Load into separate struct for security
  user := UserDTO{
    Name: u.Name,
    Email: u.Email,
    IsAdmin: false // avoids exposing field that should not be bound
  }

  executeSomeBusinessLogic(user)

 // ...
})
```

#### JSON Data

```sh
curl -X POST http://localhost:1323/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Joe","email":"joe@labstack"}'
```

#### Form Data

```sh
curl -X POST http://localhost:1323/users \
  -d 'name=Joe' \
  -d 'email=joe@labstack.com'
```

#### Query Parameters

```sh
curl -X GET 'http://localhost:1323/users?name=Joe&email=joe@labstack.com'
```

## Fluent Binding

This package provides an interface to bind explicit data types from a specified source. It uses method chaining, also known as a [Fluent Interface](https://en.wikipedia.org/wiki/Fluent_interface).

The following methods provide a handful of methods for binding to Go data type. These binders offer a fluent syntax and can be chained to configure & execute binding, and handle errors.

-   `binding.QueryParamsBinder(req)` - binds query parameters (source URL)
-   `binding.PathParamsBinder(req)` - binds path parameters (source URL)
-   `binding.FormFieldBinder(req)` - binds form fields (source URL + body). See also [Request.ParseForm](https://golang.org/pkg/net/http/#Request.ParseForm).

### Error Handling

A binder is usually completed by calling `BindError()` or `BindErrors()`. If any errors have occurred, `BindError()` returns the first error encountered, while`BindErrors()` returns all bind errors. Any errors stored in the binder are also reset.

With `FailFast(true)` the binder can be configured to stop binding on the first error, or with `FailFast(false)` execute the entire binder call chain. Fail fast is enabled by default and should be disabled when using `BindErrors()`.

### Example

```go
// url =  "/api/search?active=true&id=1&id=2&id=3&length=25"
var opts struct {
  IDs []int64
  Active bool
}
length := int64(50) // default length is 50

// creates query params binder that stops binding at first error
err := binding.QueryParamsBinder(req).
  Int64("length", &length).
  Int64s("ids", &opts.IDs).
  Bool("active", &opts.Active).
  BindError() // returns first binding error
```

### Supported Data Types

| Data Type           | Notes                                                                                                          |
| ------------------- | -------------------------------------------------------------------------------------------------------------- |
| `bool`              |                                                                                                                |
| `float32`           |                                                                                                                |
| `float64`           |                                                                                                                |
| `int`               |                                                                                                                |
| `int8`              |                                                                                                                |
| `int16`             |                                                                                                                |
| `int32`             |                                                                                                                |
| `int64`             |                                                                                                                |
| `uint`              |                                                                                                                |
| `uint8/byte`        | Does not support `bytes()`. Use `BindUnmarshaler`/`CustomFunc` to convert value from base64 etc to `[]byte{}`. |
| `uint16`            |                                                                                                                |
| `uint32`            |                                                                                                                |
| `uint64`            |                                                                                                                |
| `string`            |                                                                                                                |
| `time`              |                                                                                                                |
| `duration`          |                                                                                                                |
| `BindUnmarshaler()` | binds to a type implementing BindUnmarshaler interface                                                         |
| `TextUnmarshaler()` | binds to a type implementing encoding.TextUnmarshaler interface                                                |
| `JsonUnmarshaler()` | binds to a type implementing json.Unmarshaler interface                                                        |
| `UnixTime()`        | converts Unix time (integer) to `time.Time`                                                                    |
| `UnixTimeMilli()`   | converts Unix time with millisecond precision (integer) to `time.Time`                                         |
| `UnixTimeNano()`    | converts Unix time with nanosecond precision (integer) to `time.Time`                                          |
| `CustomFunc()`      | callback function for your custom conversion logic                                                             |

This package supported type has the following methods:

-   `<Type>("param", &destination)` - if parameter value exists then binds it to given destination of that type i.e `Int64(...)`.
-   `Must<Type>("param", &destination)` - parameter value is required to exist, binds it to given destination of that type i.e `MustInt64(...)`.
-   `<Type>s("param", &destination)` - (for slices) if parameter values exists then binds it to given destination of that type i.e `Int64s(...)`.
-   `Must<Type>s("param", &destination)` - (for slices) parameter value is required to exist, binds it to given destination of that type i.e `MustInt64s(...)`.

For certain slice types `BindWithDelimiter("param", &dest, ",")` supports splitting parameter values before type conversion is done. For example binding an integer slice from the URL `/api/search?id=1,2,3&id=1` will result in `[]int64{1,2,3,1}`.

## License

Copyright (c) 2024-present [Anurag Kumar](https://github.com/anuragkumar19)

Licensed under [MIT License](./LICENSE)

[GoDoc]: https://pkg.go.dev/github.com/anuragkumar19/binding
[GoDoc Widget]: https://godoc.org/github.com/anuragkumar19/binding?status.svg
