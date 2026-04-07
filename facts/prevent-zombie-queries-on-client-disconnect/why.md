### Why is this a massive problem?

When a standard HTTP request is made, the browser opens a TCP connection. When the user hits F5, the browser forcefully closes this socket.

1. **The Network layer knows**: The Operating System's network layer instantly recognizes the socket was dropped.
2. **The App layer is deaf**: Traditional Express.js/PHP/Java backends don't check if the socket is alive before executing the next line of code. They just keep `await`-ing the database to perform complex table scans.
3. **Ghost Processing**: The Database eventually finishes the heavy query and returns the huge payload to the backend. The backend prepares the JSON, tries to send it back via HTTP, and *only then* realizes the connection is dead (throwing an `EPIPE` or `Socket closed` error).

This means you just wasted 100% of your computation cost and database IO for a user that threw away the result 10 seconds ago. This architecture flaw is a primary vector for **Layer 7 Application DoS attacks**.
