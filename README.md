# systemd-docker-nginx

Run nginx inside a docker container on systemd.

## Installation

1. Decide where you want to install things.

   ```sh
   INSTALL_DIR=/var/lib/nginx/systemd
   ```

1. Clone this repository.

   ```sh
   $ git clone https://github.com/agross/systemd-docker-nginx.git "$INSTALL_DIR"
   Cloning into 'systemd-docker-nginx'...
   ```

1. Install `nginx` systemd service unit.

   ```sh
   $ "$INSTALL_DIR/install"
   Linking and enabling /var/lib/nginx/systemd/nginx.service
   Created symlink /etc/systemd/system/nginx.service → /var/lib/nginx/systemd/nginx.service.
   Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /var/lib/nginx/systemd/nginx.service.
   ```

1. Tell the nginx container about host-mounted paths for configuration, logs and HTML files.

   `systemd-docker-nginx` runs the nginx docker container with some required
   arguments, but you can also add `docker run` command line arguments of your
   own. Please see
   [`nginx.service.conf`](https://github.com/agross/systemd-docker-nginx/blob/master/nginx.service.conf)
   for details.

   ```sh
   $ systemctl edit nginx.service

   # Editor opens. Add these lines and save the file.

   [Service]
   EnvironmentFile=/path/to/your/custom/nginx.service.conf
   ```

1. Start nginx.
   ```sh
   $ systemctl start nginx
   ```

## Optional configuration items to think about

### logrotate

Depending on where your place nginx log files, you want to enable log rotatation. See `NGINX_LOGS` in
your `EnvironmentFile=`.

```sh
# Get the nginx log directory via EnvironmentFile=.
$ LOGS="$(systemctl show nginx.service | \
          command grep --perl-regexp --only-matching 'EnvironmentFile=\K.*(?= \(ignore_errors=)' | \
          xargs --no-run-if-empty grep --perl-regexp --only-matching 'NGINX_LOGS=\K.*')"
$ echo $LOGS
/var/data/nginx/log

# Create logrotate script.
$ cat <<EOF > /etc/logrotate.d/nginx
$LOGS/*.log {
  daily
  dateext
  missingok
  rotate 7
  compress
  delaycompress
  notifempty
  create
  sharedscripts
  postrotate
    if systemctl is-active nginx.service > /dev/null 2>&1; then
      # Make nginx re-open its log files.
      systemctl reload nginx.service
    fi
  endscript
}
EOF
```

### netdata

If you run [netdata](https://github.com/firehol/netdata), it's recommended to
start netdata after nginx such that netdata can detect nginx.

```sh
$ systemctl edit netdata.service

# Editor opens. Add these lines and save the file.

[Unit]
After=nginx.service
```

### Get notified when nginx fails

You can tell systemd to start another unit whenever a unit fails. See
[this blog post](http://northernlightlabs.se/systemd.status.mail.on.unit.failure)
for how it is done. Also see my
[systemd-unit-status-mail](https://github.com/agross/systemd-unit-status-mail)
repository that contains everything you need.

After `unit-status-mail` is in place, add it to your `nginx.service` using a
systemd override:

```sh
$ systemctl edit nginx.service

# Editor opens. Add these lines and save the file.

[Unit]
OnFailure=unit-status-mail@%n.service
```

### Getting the whole picture

To get all `nginx.service` settings use

* `systemctl cat nginx.service` and
* `systemctl show nginx.service`.
