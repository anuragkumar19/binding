# binding

[![GoDoc Widget]][GoDoc]

`binding` is a go modules which assist in binding request data to struct. The source code is taken from [Echo](https://github.com/labstack/echo) and modified to support std lib and chi.

### Few things you should know

-   URL Params binding using struct tags is only supported for chi. (not std lib)
-   Echo bind both query-form and body-form in `form` tag. But in this version it only bind body when using `form` struct tag. Use `query` tag if you want to bind query params.
-   Echo doesn't bind query for method other than `GET`, `HEAD` and `DELETE` but as we not don't bind query-form in form tag, query will be binded for all methods.

## License

Copyright (c) 2024-present [Anurag Kumar](https://github.com/anuragkumar19)

Licensed under [MIT License](./LICENSE)

[GoDoc]: https://pkg.go.dev/github.com/anuragkumar19/binding
[GoDoc Widget]: https://godoc.org/github.com/anuragkumar19/binding?status.svg
