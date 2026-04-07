When running the `restart` command, the old Nginx process will be killed immediately (along with all open connections) and a new process will start, causing a 502/Connection Refused interruption error.

When running the `reload` command:
1. The Nginx Master process re-reads the new configuration.
2. It checks the configuration for syntax errors. If there are errors, it continues keeping the old config and reports the error without affecting the system.
3. If the configuration is correct, the Master process will spawn new Worker processes using the new configuration.
4. It sends a "graceful shutdown" signal to the old Workers, instructing them to stop accepting new connections, but to continue serving current connections until they are completely finished.
5. Once the old connections are handled, the old Workers automatically close down. Throughout this entire process, site visitors experience absolutely no downtime.
