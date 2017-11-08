# JWT Middleware for Gin Framework

This is a middleware for [Gin](https://github.com/gin-gonic/gin) framework.

It uses [jwt-go](https://github.com/dgrijalva/jwt-go) to provide a jwt authentication middleware. It provides additional handler functions to provide the `login` api that will generate the token and an additional `refresh` handler that can be used to refresh tokens.

# Reason for Fork

The existing library does not update the creation time of the token when refreshed. Therefore refresh only extends the expire time of the new token.

This fork updates the refresh handler to extend the refresh time of the new token. This change allows for the new token generated from the refresh to be refreshed for the `MaxRefresh` time to retrieve another token. More simply this change will allow for a user's authentication to be extended indefinitely with consistent refreshes (refreshing before `MaxRefresh` timeout since last refresh/login).

## Usage

Download and install it:

```sh
$ go get github.com/dataBaseError/gin-jwt
```

Import it in your code:

```go
import "github.com/dataBaseError/gin-jwt"
```

## Example

Please see [the example file](example/server.go) and you can use `ExtractClaims` to fetch user ID.

[embedmd]:# (example/server.go go)
```go
package main

import (
	"net/http"
	"os"
	"time"

	"github.com/dataBaseError/gin-jwt"
	"github.com/gin-gonic/gin"
)

func helloHandler(c *gin.Context) {
	claims := jwt.ExtractClaims(c)
	c.JSON(200, gin.H{
		"userID": claims["id"],
		"text":   "Hello World.",
	})
}

func main() {
	port := os.Getenv("PORT")
	r := gin.New()
	r.Use(gin.Logger())
	r.Use(gin.Recovery())

	if port == "" {
		port = "8000"
	}

	// the jwt middleware
	authMiddleware := &jwt.GinJWTMiddleware{
		Realm:      "test zone",
		Key:        []byte("secret key"),
		Timeout:    time.Hour,
		MaxRefresh: time.Hour,
		Authenticator: func(userId string, password string, c *gin.Context) (string, bool) {
			if (userId == "admin" && password == "admin") || (userId == "test" && password == "test") {
				return userId, true
			}

			return userId, false
		},
		Authorizator: func(userId string, c *gin.Context) bool {
			if userId == "admin" {
				return true
			}

			return false
		},
		Unauthorized: func(c *gin.Context, code int, message string) {
			c.JSON(code, gin.H{
				"code":    code,
				"message": message,
			})
		},
		// TokenLookup is a string in the form of "<source>:<name>" that is used
		// to extract token from the request.
		// Optional. Default value "header:Authorization".
		// Possible values:
		// - "header:<name>"
		// - "query:<name>"
		// - "cookie:<name>"
		TokenLookup: "header:Authorization",
		// TokenLookup: "query:token",
		// TokenLookup: "cookie:token",

		// TokenHeadName is a string in the header. Default value is "Bearer"
		TokenHeadName: "Bearer",

		// TimeFunc provides the current time. You can override it to use another time value. This is useful for testing or if your server uses a different time zone than your tokens.
		TimeFunc: time.Now,
	}

	r.POST("/login", authMiddleware.LoginHandler)

	auth := r.Group("/auth")
	auth.Use(authMiddleware.MiddlewareFunc())
	{
		auth.GET("/hello", helloHandler)
		auth.GET("/refresh_token", authMiddleware.RefreshHandler)
	}

	http.ListenAndServe(":"+port, r)
}
```

## Demo

Please run example/server.go file and listen `8000` port.

```bash
$ go run example/server.go
```

Download and install [httpie](https://github.com/jkbrzt/httpie) CLI HTTP client.

### Login API:

```bash
$ http -v --json POST localhost:8000/login username=admin password=admin
```

Output screenshot

![api screenshot](screenshot/login.png)

### Refresh token API:

```bash
$ http -v -f GET localhost:8000/auth/refresh_token "Authorization:Bearer xxxxxxxxx"  "Content-Type: application/json"
```

Output screenshot

![api screenshot](screenshot/refresh_token.png)

### Hello world

Please login as `admin` and password as `admin`

```bash
$ http -f GET localhost:8000/auth/hello "Authorization:Bearer xxxxxxxxx"  "Content-Type: application/json"
```

Response message `200 OK`:

```
HTTP/1.1 200 OK
Content-Length: 24
Content-Type: application/json; charset=utf-8
Date: Sat, 19 Mar 2016 03:02:57 GMT

{
    "text": "Hello World."
}
```

### Authorization

Please login as `test` and password as `test`

```bash
$ http -f GET localhost:8000/auth/hello "Authorization:Bearer xxxxxxxxx"  "Content-Type: application/json"
```

Response message `403 Forbidden`:

```
HTTP/1.1 403 Forbidden
Content-Length: 62
Content-Type: application/json; charset=utf-8
Date: Sat, 19 Mar 2016 03:05:40 GMT
Www-Authenticate: JWT realm=test zone

{
    "code": 403,
    "message": "You don't have permission to access."
}
```
