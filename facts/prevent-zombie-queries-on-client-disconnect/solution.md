Never let your backend run queries blindly into the void. You must strictly connect the **Request Lifecycle** to the **Database Execution Context**.

The exact solution is using an **AbortSignal** to cascade the cancellation intent from the network layer all the way down to the database driver. 

1. **Detect**: The web framework detects when the TCP connection is unexpectedly dropped by the client.
2. **Propagate**: It triggers a cancellation token (usually a standard `AbortSignal`).
3. **Terminate**: The database driver (e.g., PostgreSQL, Mongoose, or an upstream API) listens to this token and immediately aborts the SQL query execution, freeing up server resources instantly.
