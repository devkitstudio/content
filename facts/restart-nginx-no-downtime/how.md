1. Modify the Nginx configuration in `/etc/nginx/nginx.conf` or `/etc/nginx/conf.d/...`
2. Check the syntax for errors before reloading:
```bash
nginx -t
```
3. If the output says `syntax is ok`, reload the new configuration without downtime:
```bash
nginx -s reload
```
