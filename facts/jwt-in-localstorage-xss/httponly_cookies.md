The industry standard solution is to instruct your Backend to place the JWT inside an **HttpOnly Cookie**, NOT in the JSON payload body.

### How it works
When the user logs in, your Backend returns a `Set-Cookie` header:
```http
Set-Cookie: token=eyJhbG...; HttpOnly; Secure; SameSite=Strict;
```

### Why is this invincible to XSS?
Because of the `HttpOnly` flag! 
`HttpOnly` literally means: *"Hey Browser, do not let ANY JavaScript read this cookie."*

If the same hacker from before tries to inject malicious JavaScript and runs `document.cookie`, the cookie won't be there. It is completely invisible to JavaScript. However, whenever your React app sends a `fetch()` request to your API, the browser automatically attaches the cookie under the hood!
