http-echo
=========
HTTP Echo is a small go web server that serves the contents it was started with
as an HTML page.

The default port is 5678, but this is configurable via the `-listen` flag:

```
http-echo -listen=:8080 -text="hello world"
```

Then visit http://localhost:8080/ in your browser.

Run locally :
```
make docker && docker run --rm -p 5678:5678 -e ECHO_TEXT="Hello Docker" http-echo:local
```

Test locally:
```
curl http://localhost:5678

```