+++
title = "Systemd Service Units"
date = '2026-01-11T19:03:05Z'
draft = false
tags = ["systemd-units","rhcsa"]
series = ["systemd"]
categories = ["tutorials"]
+++

## Introduction

Systemd service units are files which define a service, and when/how it's started. They will normally refer to a program or script to be executed. There are 3 fundamental sections on a service file:

- `[Unit]` is where the description resides, as well as the conditions to initialize the service, such as requirements and dependencies.
  
- `[Service]` is where the parameters which determine how to run the service reside.
  
- `[Install]` is how to enable the unit. It's optional, but normally necessary.
  

## The `[Unit]` Section

Let's go through some fundamental fields and what goes in them:

- `Description=` A human-readable description. Example: `Description=Apache Web Server`.
  
- `After=` It determines after which units this unit will be started. Example: `After=network.target`.
  
- `Before=` It determines before which units this unit will be started. Example: `Before=httpd.service`.
  
- `Requires=` It determines a hard dependency, without which this unit fails. Example: `Requires=network.target`.
  
- `Wants=` This is a soft dependency. It should be started before this unit, but not required. Example: `Wants=network-online.target`.
  
- `BindsTo=` This is a stronger dependency than `Requires=`. Stops if this dependency stops. Example: `BindsTo=dev-sda1.device`.
  

To summarize, there are fields which determine ordering: `After=` & `Before=`, and there are fields which determine dependency: `Requires=`, `Wants=` & `BindsTo=`.

## The `[Service]` Section

These are the most usual fields which determine how the service:

- `Type=` How Systemd tracks the service. Example: `Type=simple`.
  
- `ExecStart=` Command used to start the service. Example: `ExecStart=/usr/sbin/httpd - DFOREGROUND`
  
- `ExecStop=` Command to stop. This is optional, `SIGTERM` is default. Example: `ExecStop=/usr/sbin/httpd -k stop`.
  
- `ExecReload=` Command to reload configuration. Example: `ExecReload=/bin/kill -HUP $MAINPID`
  
- `Restart=` When to auto-restart. Example: `Restart=on-failure`
  
- `RestartSec=` Delay before attempting to restart. Example: `RestartSec=5`
  
- `User=` Run this service as this user. Example: `User=apache`
  
- `Group=` Run this service as this user. Example: `Group=apache`
  
- `WorkingDirectory=` Sets the working directory for the service. Example: `WorkingDirectory=/var/www`
  
- `Environment=` Set environment variable. Example: `Environment=LANG=en_US.UTF-8`
  
- `EnvironmentFile=` Load environment variables from file. `EnvironmentFile=/etc/sysconfig/httpd`
  
- `StandardOutput=` Where to send `STDOUT`. Example: `StandardOutput=journal`
  
- `StandardError=` Where to send `STDERR`. Example: `StandardError=journal`
  

### Service Types

These are possible values which go in the `Type=` field:

- `simple` This is the default. Process started by `ExecStart=` is the main process. This is used when the service runs in the foreground, doesn't fork.
  
- `forking` Process forks into background. Systemd waits for parent to exit. Tradition daemons that fork. Use `PIDFile=`.
  
- `oneshot` Process runs and exits. Service is `active` while running. This is for scripts that do one task and exit.
  
- `notify` Like `simple`, but service signals readiness via `sd_notify()`. This is for services built with Systemd notification.
  
- `dbus` Service is ready whenit acquires its D-Bus name. D-Bus activated services.
  
- `idle` Like `simple`, but waits until other jobs finish. This is for low-priority startup tasks.
  

### Restart Policies

These are possible values which go in the `Restart=` field:

- `no` Never. This is default.
  
- `on-success` Clean exit (exit code 0).
  
- `on-failure` Non-zero exit, signal, timeout, watchdog.
  
- `on-abnormal` Signal, timeout, watchdog (not exit codes).
  
- `on-abort` Unclean signal (`SIGABRT`, `SIGSEGV`, etc).
  
- `on-watchdog` Watchdog timeout only.
  
- `always` Always, regardless of exit reason.
  

## The `[Install]` Section

The `[Install]` section tells systemd what happens when you run `systemctl enable`. It doesn't affect runtime behavior, but only controls where symlinks get created:

- `WantedBy=` Target that "wants" this unit (creates symlink). Think of it as "start my service when this target activates, but if it fails, the target still succeeds". Example: `WantedBy=multi-user.target`.
  
- `RequiredBy=` Target that "requires" this unit. Think of it as "start my service when this target activates, and if it fails, the target fails too". Example: `RequiredBy=graphical.target`.
  
- `Alias=` Alternative names for the unit. Example: `Alias=www.service`.
  

**How `enable` works:** When you run `systemctl enable myservice`, Systemd reads the `[Install]` section and creates a symlink:

```bash
/etc/systemd/system/multi-user.target.wants/myservice.service -> /etc/systemd/system/myservice.service
```

*Vendor units live in `/usr/lib/systemd/system`*

If there is no `[Install]` section, the unit can't be enabled/disabled, but runs only as a dependency.

## Minimal Service Template

```ini
[Unit]
Description=My Custom Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myscript.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## Follow Along Example

Let's set up a simple service to make sure that the concepts stick. First, write a simple shell script which just does something. If you want the easy way, as `root`, copy and paste the following heredoc:

```bash
cat > /usr/local/bin/myservice.sh << 'EOF'
#!/bin/bash
while true; do
    echo "$(date): Running" >> /tmp/myservice.log
    sleep 10
done
EOF
chmod +x /usr/local/bin/myservice.sh
```

Test if your script works by running it with `/usr/local/bin/myservice.sh`. It will make the terminal busy. Press `CTRL+C` to stop it and check like this:

```bash
[root@server1 ~]# cat /tmp/myservice.log 
Sun Jan 11 06:42:17 PM GMT 2026: Running
```

If it worked like in this example, you're gold. Next, choose your editor and create a unit file for `myservice.sh`. In my case I prefer vim (you should too): `vim /etc/systemd/system/myservice.service`:

```ini
[Unit]
Description=A simple service which logs time every 10 sec
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myservice.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Alternatively, if you're feeling lazy, copy and paste the heredoc:

```bash
cat > /etc/systemd/system/myservice.service << 'EOF'
[Unit]
Description=A simple service which logs time every 10 sec
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myservice.sh
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF
```

To make sure that Systemd recognizes the service, run:

1. `systemctl daemon-reload`
  
2. `systemctl start myservice.service`
  
3. `systemctl status myservice.service` to see that it's running.
  
4. If you want it to enable it to always run when booting: `systemctl enable myservice.service`.
  

You could check that the service is running in two ways:

- `tail -f /tmp/myservice.log` and you will see a new line every 10 seconds.
  
- `journalctl -u myservice -f` (this is fancier) and you should see logs coming up every 10 seconds, plus the history of the service.
  

To stop it, run `systemctl stop myservice.service`.

## Conclusion

In this lesson you have learned what the basic components of a Systemd service unit file are, and how to create a `simple` service. I hope that this was instructive. More on Systemd will follow. Next week I hope to post about Systemd timers. Thank you for your read!