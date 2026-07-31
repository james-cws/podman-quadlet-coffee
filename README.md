# Podman Quadlet for 418

This is an example quadlet showing a *very* high degree of isolation for
an nginx web server, using error code 418 for funsies.

This requires a reverse proxy in front of it to handle forwarding the
incoming connection. The only requirements for the reverse proxy are to be
able to connect to a Unix socket, and to be on the same physical host
machine (we're using file based access only here).

While this is truly excessive for a static page informing users that
coffee cannot be served for reason: am teapot, it's a fun way to show how
I handle a high degree of isolation for a web server and this can be
adapted/extended to apply to a wide range of web exposed services.

## Architectural Description

### Reverse proxy

A reverse proxy needs to sit in front of this server to handle forwarding
the incoming connections. This is required for any network access at all.

Here's an example snippet for Nginx, but any web server that can connect
to a Unix socket may be used:

```
	server {
		listen	[::]:80;
		listen	80;
		server_name $name_http;
		location / {
			proxy_pass http://$name_http;

			proxy_set_header Host $host;
			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header X-Forwarded-Proto $scheme;
		}

		error_page 500 502 503 504 /50x.html;
		location = /50x.html {
			root /usr/share/nginx/html/;
		}
	}

	upstream coffee_backend_http {
		server unix:/sockets/coffee/coffee.sock;
	}
```

And then the reverse proxy quadlet would also need in its quadlet's
`[Container]` section:

```
Volume=/run/user/1234/proxy:/sockets/coffee:z
```

Where `1234` is the UID of the unprivileged user account owning our
example teapot container. I prefer to keep sockets in `/run` or
`/var/run` but (with appropriate permissions set) any directory can be
used. One immediate advantage (aside from it being common practice to put
sockets under `/run` or `/var/run`) is that the user's run directory has
additional builtin protections that help prevent unauthorized access to
the socket where the socket file itself is wide open to allow the reverse
proxy's forked web user to read/write to the socket.

### Web server

The whole raison d'être of this little project is to be a proof of
concept/demo of a highly isolated, unprivileged (rootless) container that
provides a web server, so there are choices here that make dev/debug more
difficult than they strictly need to be.

The first level of isolation is, since the reverse proxy handles opening
ports and network access, we don't need root (reverse proxy can be
rootful or rootless, but this demo assumes it's rootful and serving on
port 80; rootless requires some additional permissions sorcery). So we
create a completely unprivileged user that can own this container.

The next level is to change the container namespace using `UserNS=auto`,
which is helpful if we want to, for example, run a couple of pods under
this user but don't necessarily want them to be able to talk to each
other (typically better to put each pod under its own user, but we're
doing things the hard way here).

Then, since the connection is file based only we don't need networking
so `Network=none` will prevent the container from reaching out. Note
that this also means no access to the container via `localhost` either
(but still accessible locally, eg using
`curl --unix-socket /run/user/1234/proxy/coffee.sock http://serverip/`).

What this accomplishes is the web server that clients connect to is
thoroughly unprivileged; for a more complex web app this hinders a bad
actor who has compromised the web app or container from moving laterally
through the machine or network. **This does not make the container
un-hackable or inescapable**. It just makes it harder if/when the
container does get compromised.

### Areas to improve upon

For a real/production deployment, there are a few things that should be
fixed up:

1. Nginx configurations and html code would be better if built into the
   container directly, not volume mounted. **IMPORTANT** while this is
   better, make sure some sort of CI/CD plan is in place to ensure your
   new custom image is being kept up to date with whatever base image you
   choose to use. In this example we're not modifying the base nginx
   image so we can auto update when new releases are pushed to the repo.

2. SSL isn't covered here; use it!!! In this architecture the reverse
   proxy would be the place to do it. Images like Caddy make it pretty
   easy.

## Getting started

1. Create a new user for the rootless container (requires root/admin):

   `sudo useradd teapot`

2. Enable linger to allow running services while not logged in:

   `sudo loginctl enable-linger teapot`

3. Start a shell as the user:

   `sudo machinectl shell teapot@`

4. As the new user `teapot` clone this repo:

   `git clone https://gitlab.com/james-cws/podman-quadlet-coffee.git`

5. Setup our directory structure in the new user's home:

   `mkdir -p ~/.config/containers/systemd ~/.config/nginx ~/.local/share/nginx

6. Install the quadlet unit files:

   `cp podman-quadlet-coffee/quadlets/* ~/.config/containers/systemd/`

7. Using your favourite text editor, edit/change at least the nginx
   configuration file to your own, owned domain

   `vim ~/.config/nginx/00_418.conf`

8. Reload the daemon & start the pod:

   `systemctl --user daemon-reload`
   `systemctl --user start coffee_418.service`

9. Check the logs for errors:

   `journalctl --user -xe`

10. Enable auto updates:

   `systemctl --user enable --now podman-auto-update.timer`

## Known Issues

1. Startup ordering isn't covered; this is a little tricky & I haven't
   devised a general case solution yet. I'll update when I have something
