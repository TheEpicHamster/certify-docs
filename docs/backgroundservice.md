---
id: backgroundservice
title: Background Service
---

In order to perform certificate requests and automatic renewals we install a background service called "Certify.Service" (Certify Certificate Manager Service). This service is installed to run as Local System and requires that the Local System account has the necessary privileges to administer IIS (if required) and the computers certificate store, as well as writing to the C:\ProgramData\Certify folder for configuration information. For more information on security and required permissions see [security](guides/security.md)

To check the log for this service, review `C:\ProgramData\Certify\logs\service.exceptions.log`.

## Port 9696 is the default service port

By default the background service runs a local http API server on port 9696 for the UI to talk to (bound to localhost and requiring windows authentication). _Do not open this port to external traffic on your firewall._

## Custom configuration and Troubleshooting "..service not started" error

The certify background service operates a local API for the app on port `9696` by default. If this port is in use by another application/service (or for some other reason it cannot create a binding to `localhost:9696`, or a security product is preventing **local** port access) then you will see the message 'Service not started'.

:::info
If you are repeatedly seeing the "Service Not Started" error, first try deleting `serviceconfig.json` and `servers.json` from C:\ProgramData\Certify\ then restart the background service and the app. This can help if automatic port negotiation has gotten out of sync.
:::

The app should try to negotiate a different service port if it detects that the port is already in use, however you can manually specify the settings if required by editing/adding the file `c:\programdata\certify\serviceconfig.json` with content as per the following (adjusted as required) then restarting both the service and UI:

```json
{
  "host": "localhost",
  "port": 9696
}
```

For example an alternative configuration might be:

```json
{
  "host": "127.0.0.1",
  "port": 9695
}
```

You may also need to update corresponding configuration in the `servers.json` file (which the UI refers to in order to locate the service).

To test that the reconfigured service is communicating OK, you can try opening the following URL in your browser:
`http://localhost:9695/api/system/appversion` where 'localhost' is your configured service `host` value and `9695` is an example configured port.

You can also try the same using PowerShell:

```ps
Invoke-RestMethod -Uri http://localhost:9696/api/system/appversion -UseDefaultCredentials
```

## Other Considerations for 'Service Not Started..'

To operate properly the background service needs to be able to register an http listener for it's API server via http.sys, for that to work the IP address the service tries to use must be enabled as an http listen address and in some versions of windows the Http kernel service may not be enabled and you will need to enable it.

### Allow Local System account to bind an http listener to the service port

In some cases you need to explicitly allow the service to listen as an http service on the localhost IP address. To Do so run the following command from the command line as an Administrator:

`netsh http add urlacl url=http://127.0.0.1:9696/ user="NT AUTHORITY\SYSTEM"`

### Enable http listener IP address

As per https://docs.microsoft.com/en-us/windows/win32/http/add-iplisten enable any IP address to listen for http:

```bat
netsh http add iplisten ipaddress=0.0.0.0
```

Or to target a specific IP address such as 127.0.0.1 (localhost):

```bat
netsh http add iplisten ipaddress=127.0.0.1
```

By default the windows http service is typically enabled but if you receive the error 'Operation is not supported on this platform' in `service.exceptions.log` then try checking the status of the windows http service. To do so, run the following from an elevated command prompt (using Run As Administrator):

```bat
sc query http
```

This should produce output like:

```bat
SERVICE_NAME: http
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 4  RUNNING
                                (STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0

```

If the state is not `RUNNING` use the following command the enable the service on demand:

```bat
sc config http start= demand
```

Then start the http service

```bat
net start http
```

If the service remains at `STOPPING` or similar then a system reboot may be required.

Once completed, restart the Certify background service from local services, then open the Certify The Web UI again to see if it can connect.

## Managed Items Database

The data store for the managed certificates database is the C:\ProgramData\Certify\manageditems.db SQLite database. This stores your renewal configuration data (not certificates). This is an sqlite3 format database files.

You should include C:\ProgramData\Certify\ in your normal backup procedures, otherwise if you lose this configuration or it is corrupted you may need to add all of your managed certificates again. You may consider adding an exclusion to your anti-virus software to avoid sharing conflicts.

On service startup and on a daily schedule the system will make a copy of manageditems.db called manageditems.db.bak.

### Data Recovery

Storage write errors or sudden unexpected system shutdowns can (in rare occasions) cause database corruption. In the event that your database (and backup) become corrupted the sqlite tools may be used to recover some or all of the information.

### Attempt an export

The following will copy the contents of manageditems.db to a new db file. You can try this in place of your original:

```
sqlite3.exe manageditems.db
sqlite> .backup output.db
```

Then copy new.db to manageditems.db and start the Certify service.

### Attempt a recovery using SQL dump

Based on the example from https://ronnieroller.com/Repair-an-SQLite-database we can export the database

```
sqlite3 manageditems-corrupted.db
sqlite> .mode insert
sqlite> .output dump.sql
sqlite> .dump
sqlite3.exe new.db < dump.sql
```

```
sqlite3 new.db "PRAGMA integrity_check"
```

Then copy new.db to manageditems.db and start the Certify service.
