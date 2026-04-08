Instead of cluttering your controller files with endless `if (version === 'v2')` conditionals, the industry standard is to separate version logic using an API Gateway routing pattern.

This pattern maps the requested version directly to a dedicated directory of handlers, isolating codebase changes from legacy versions.

```javascript
// Use API gateway to route to correct handler
const apiGateway = {
  'v1': require('./handlers/v1'),
  'v2': require('./handlers/v2'),
  'v3': require('./handlers/v3')
};

app.get('/api/:version/users/:id', (req, res, next) => {
  const { version, id } = req.params;
  const handler = apiGateway[version];

  if (!handler) {
    return res.status(404).json({ error: 'API version not found' });
  }

  handler.getUser(id, (err, user) => {
    if (err) return next(err);
    res.json(user);
  });
});

// Directory structure:
// handlers/
//   v1/
//     users.js
//     products.js
//   v2/
//     users.js
//     products.js
//   v3/
//     users.js
//     products.js
```
