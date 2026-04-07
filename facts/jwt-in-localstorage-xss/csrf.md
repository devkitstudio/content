When you switch from LocalStorage to Cookies, you solve **XSS**, but you open the door to a new attack vector: **CSRF (Cross-Site Request Forgery).**

### What is CSRF?
Because the browser *automatically* attaches cookies to requests, if a hacker tricks your user into clicking a malicious link (`https://evil.com/auto-submit-form`), the browser will blindly attach your banking token cookie to that malicious request!

### How to defeat CSRF?
1. **SameSite=Strict**: Always configure your cookie with the `SameSite: Strict` attribute. This stops the browser from attaching the cookie if the request originates from a different domain.
2. **CSRF Tokens**: Your backend can generate a short-lived anti-CSRF token on every request, verifying it before processing any mutations.

**The Golden Rule:** XSS (stealing your tokens) is catastrophically worse than CSRF (making unwanted automated requests). Therefore, defending against XSS with HttpOnly cookies is always the priority.
