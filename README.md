# Cloud Engineering Vorbereitung: The Simple Web Server

## Aufgabe 1: Neues Projekt erstellen
Installieren Sie [Go](https://go.dev/), [Git](https://git-scm.com/) und entsprechende Tools wie
[Language-Server](https://pkg.go.dev/golang.org/x/tools/gopls) und
[Formatter](https://pkg.go.dev/cmd/gofmt). Vergessen Sie nicht, entsprechende
Verzeichnisse zu den Umgebungsvariablen hinzuzufügen.

Erstellen Sie nun ein neues Verzeichnis für das Projekt.

```bash
mkdir simple-web-server && cd simple-web-server
```

Initialisieren Sie als nächstes das Repository mit
[git-init](https://git-scm.com/docs/git-init) und benennen Sie den Branch von
`master` auf `main` um.

```bash
git init && git branch -m main
```

Erstellen Sie das Go-Module mit [go-mod](https://go.dev/ref/mod) im selben Verzeichnis.

```bash
go mod init github.com/<github-account>/simple-web-server
```

Abschließend erstellen Sie im Hauptverzeichnis des Projekts die Datei `main.go` mit folgendem Inhalt.

```go
package main

func main() {
}
```

Die Projektstruktur sollte nun wie folgt aussehen.

```
.
├── .git
├── go.mod
└── main.go
```

## Aufgabe 2: Webserver
Erstellen Sie nun eine Serveranwendung mit Go. Der Server soll eine Route `/`
behandeln. Bei einer Anfrage gegen die Route soll eine HTML-Seite mit der
Überschrift `Hello Web!` ausgeliefert werden. Falls ein `GET`-Request gegen
einen anderen Pfad gesendet wird, soll eine HTML-Seite mit der Überschrift `404
NOT FOUND` gerendert und zusätzlich `404` als Statuscode gesetzt werden. In
allen anderen Fällen soll keine Seite gerendert werden, jedoch soll mit `404`
als Statuscode geantwortet werden. **Schreiben Sie außerdem sinnvolle Tests,
die Ihre Anwendung komplett abtesten. Grundsätzlich gilt, dass manuelle Tests
(z.B. durch Navigieren über den Browser) überflüssig sein sollen**. Sie können
[diesen Beitrag](https://golang.cafe/blog/golang-httptest-example.html) als
Hilfestellung verwenden.

* Für den Webserver verwenden Sie bitte `net/http` aus der Standardbibliothek.
  Für HTTP-Servertests: `net/http/httptest`.
* Für das Rendern und Laden der Templates verwenden Sie bitte `html/template`
  aus der Standardbibliothek.
* Als Testframework nutzen Sie `testing` aus der Standardbibliothek.
* Für Assertions können Sie `github.com/stretchr/testify/assert` als externe
  Abhängigkeit verwenden.
* Falls Sie Mocks nutzen möchten, können Sie `go.uber.org/mock/gomock` als
  externe Abhängigkeit verwenden. Mocks können Sie mithilfe von
  [gomock](https://github.com/uber-go/mock) automatisiert erzeugen lassen.

**Beispiele für Request/Responses:**

| Request | Statuscode | Inhalt |
| ------- | ---------- | ------ |
|`GET /`  | `200 OK` | Template `index.tmpl` rendern und zurückgeben |
|`GET /test` | `404 NOT FOUND` | Template `404.tmpl` rendern und zurückgeben |
|`POST /` | `404 NOT FOUND` | kein Inhalt |
|`POST /test` | `404 NOT FOUND` | kein Inhalt |

<details>
    <summary>Lösungsvorschlag anzeigen</summary>

_Projektstruktur_
```
.
├── .git
├── _mocks
│   └── renderer.go
├── go.mod
├── go.sum
├── main.go
├── server
│   ├── renderer.go
│   ├── router.go
│   └── router_test.go
└── templates
    ├── 404.tmpl
    └── index.tmpl
```

Hier der Vollständigkeit halber der Befehl zum Erzeugen des Mocks.

```bash
mockgen -source=server/renderer.go -package=mocks -destination=_mocks/renderer.go
```

_index.tmpl_
```html
{{ define "index" }}
<!DOCTYPE html>
<html>
    <head>
        <title>Simple Web Server</title>
    </head>
    <body>
        <h1>Hello Web!</h1>
    </body>
</html>
{{ end }}
```

_404.tmpl_
```html
{{ define "404" }}
<!DOCTYPE html>
<html>
    <head>
        <title>Simple Web Server</title>
    </head>
    <body>
        <h1>404 NOT FOUND</h1>
    </body>
</html>
{{ end }}
```

_renderer.go_
```go
package server

import "io"

type Renderer interface {
	ExecuteTemplate(w io.Writer, template string, data any) error
}
```

_router.go_
```go
package server

import (
	"net/http"
)

type Router struct {
	mux      *http.ServeMux
	renderer Renderer
}

func NewRouter(renderer Renderer) *Router {
	router := &Router{
		mux:      http.NewServeMux(),
		renderer: renderer,
	}

	router.mux.HandleFunc("/", router.handleIndex)

	return router
}

func (router *Router) handleIndex(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case "GET":
		if r.URL.Path != "/" {
			w.WriteHeader(http.StatusNotFound)
			router.renderer.ExecuteTemplate(w, "404", nil)
			return
		}

		router.renderer.ExecuteTemplate(w, "index", nil)
	default:
		w.WriteHeader(http.StatusNotFound)
	}
}

func (router *Router) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	router.mux.ServeHTTP(w, r)
}
```

_router_test.go_
```go
package server

import (
	"io"
	"net/http"
	"net/http/httptest"
	"testing"
	"text/template"

	mocks "github.com/hsfl-cloud-engineering/simple-web-server/_mocks"
	"github.com/stretchr/testify/assert"
	"go.uber.org/mock/gomock"
)

func TestRouter(t *testing.T) {
	ctrl := gomock.NewController(t)

	renderer := mocks.NewMockRenderer(ctrl)
	router := NewRouter(renderer)

	t.Run("index handler", func(t *testing.T) {
		t.Run("should return 404 NOT FOUND if request method is not GET", func(t *testing.T) {
			// given
			w := httptest.NewRecorder()
			r := httptest.NewRequest("POST", "/", nil)

			renderer.
				EXPECT().
				ExecuteTemplate(w, gomock.Any(), gomock.Any()).
				Times(0)

			// when
			router.ServeHTTP(w, r)

			// then
			res := w.Result()
			assert.Equal(t, http.StatusNotFound, res.StatusCode)
		})

		t.Run("should return 404 NOT FOUND if request method is GET but path is unknown", func(t *testing.T) {
			// given
			w := httptest.NewRecorder()
			r := httptest.NewRequest("GET", "/unknown", nil)

			renderer.
				EXPECT().
				ExecuteTemplate(w, "404", nil).
				Return(nil)

			// when
			router.ServeHTTP(w, r)

			// then
			res := w.Result()
			assert.Equal(t, http.StatusNotFound, res.StatusCode)
		})

		t.Run("should return 200 OK", func(t *testing.T) {
			// given
			w := httptest.NewRecorder()
			r := httptest.NewRequest("GET", "/", nil)

			renderer.
				EXPECT().
				ExecuteTemplate(w, "index", nil).
				Return(nil)

			// when
			router.ServeHTTP(w, r)

			// then
			res := w.Result()
			assert.Equal(t, http.StatusOK, res.StatusCode)
		})
	})
}

func TestRouterWithTemplates(t *testing.T) {
	renderer := template.Must(template.ParseGlob("../templates/*.tmpl"))
	router := NewRouter(renderer)

	t.Run("index page", func(t *testing.T) {
		t.Run("should contain greeter", func(t *testing.T) {
			// given
			w := httptest.NewRecorder()
			r := httptest.NewRequest("GET", "/", nil)

			// when
			router.ServeHTTP(w, r)

			// then
			res := w.Result()
			html, _ := io.ReadAll(res.Body)
			assert.Contains(t, string(html), "<h1>Hello Web!</h1>")
		})

		t.Run("should display 404 NOT FOUND if route is unknown", func(t *testing.T) {
			// given
			w := httptest.NewRecorder()
			r := httptest.NewRequest("GET", "/unknown", nil)

			// when
			router.ServeHTTP(w, r)

			// then
			res := w.Result()
			html, _ := io.ReadAll(res.Body)
			assert.Contains(t, string(html), "<h1>404 NOT FOUND</h1>")
		})
	})
}
```

_main.go_
```go
package main

import (
	"flag"
	"fmt"
	"html/template"
	"log"
	"net/http"

	"github.com/hsfl-cloud-engineering/simple-web-server/server"
)

func main() {
	port := flag.Int("port", 3000, "The port used to listen and serve")
	flag.Parse()

	renderer := template.Must(template.ParseGlob("templates/*.tmpl"))
	router := server.NewRouter(renderer)
	log.Fatal(http.ListenAndServe(fmt.Sprintf(":%d", *port), router))
}
```
</detail>
