# Zitadel on NixOS

Run [Zitadel](https://zitadel.com/) as a normal NixOS service. No containers. PostgreSQL is on the same machine, nginx terminates TLS, and Zitadel serves OpenID Connect and SAML 2.0.

The public URL in this example is `https://auth.example.com`. Replace that name, and the administrator email, before you use it.

This matches the `services.zitadel` module on current nixos-unstable: `tlsMode`, `extraStepsPaths`, and `services.postgresql.ensureUsers.ensureClauses`.

## What has to be true

Zitadel will answer "instance not found", or the console will not load, unless all of these hold.

- The URL users type is `https://auth.example.com` with port 443. `ExternalDomain`, `ExternalPort`, and `ExternalSecure` must be that URL. `tlsMode` is `external`, so Zitadel itself speaks plain HTTP and tells clients to use HTTPS.
- The reverse proxy forwards to Zitadel as HTTP/2 cleartext (h2c) and leaves the `Host` header alone. nginx `grpc_pass` does this. A normal `proxy_pass` does not, and both the console and the gRPC API fail behind it.
- The database password is not in the Nix store. Zitadel connects through the PostgreSQL Unix socket, and peer authentication maps the `zitadel` system user to the `zitadel` database role. That role needs `LOGIN`, `CREATEDB`, and `CREATEROLE` so the first start can create its database objects.
- The master key is 32 bytes, is created on the machine, and is never replaced. It encrypts secrets in the database. A new key cannot read a database written by the old one.
- The first administrator is created from a file under `/var/lib/zitadel`, not from a Nix option. `services.zitadel.steps` is rendered into the world-readable Nix store, and the module appends that file *after* `extraStepsPaths`, so a password set in `steps` overwrites the one on disk. Leave `steps` unset. The external file is applied only while the instance is first created. Changing it later does not reset the password.

Zitadel listens on all interfaces. Leave `services.zitadel.openFirewall` at its default, `false`, and do not publish the internal port.

## Configuration

Save this as `/etc/nixos/services/zitadel.nix`.

```nix
{ config, pkgs, lib, ... }:

let
  domain = "auth.example.com";
  adminEmail = "admin@example.com";
  listenPort = 8088;
  stateDir = "/var/lib/zitadel";
  stepsFile = "${stateDir}/secrets/steps.yaml";
in
{
  services.postgresql = {
    enable = true;
    # Pin a major version. The data directory is
    # /var/lib/postgresql/<major>, and a later major will not open it.
    package = pkgs.postgresql_17;

    ensureDatabases = [ "zitadel" ];
    ensureUsers = [
      {
        name = "zitadel";
        ensureDBOwnership = true;
        ensureClauses = {
          login = true;
          createdb = true;
          createrole = true;
        };
      }
    ];
  };

  services.zitadel = {
    enable = true;
    masterKeyFile = "${stateDir}/master.key";
    tlsMode = "external";
    extraStepsPaths = [ stepsFile ];

    settings = {
      Port = listenPort;
      ExternalDomain = domain;
      ExternalPort = 443;
      ExternalSecure = true;

      Database.postgres = {
        Host = "/run/postgresql";
        Port = 5432;
        Database = "zitadel";
        User = {
          Username = "zitadel";
          SSL.Mode = "disable";
        };
        Admin = {
          Username = "zitadel";
          SSL.Mode = "disable";
        };
      };
    };
  };

  # Creates the master key and the first administrator the first time it runs.
  # Both files stay on the machine. Nothing here is copied into the Nix store.
  systemd.services.zitadel-bootstrap = {
    description = "Create the Zitadel master key and initial administrator";
    wantedBy = [ "multi-user.target" ];
    before = [ "zitadel.service" ];
    path = [ pkgs.python3 pkgs.coreutils ];
    serviceConfig = {
      Type = "oneshot";
      RemainAfterExit = true;
    };
    script = ''
      set -euo pipefail

      install -d -m 0750 -o zitadel -g zitadel ${stateDir}
      install -d -m 0750 -o zitadel -g zitadel ${stateDir}/secrets

      key=${stateDir}/master.key
      if [ ! -f "$key" ]; then
        python3 -c 'import secrets,string; alphabet=string.ascii_letters+string.digits; print("".join(secrets.choice(alphabet) for _ in range(32)), end="")' > "$key"
        chown zitadel:zitadel "$key"
        chmod 0400 "$key"
      fi

      pwfile=${stateDir}/admin-password
      if [ ! -s "$pwfile" ]; then
        python3 -c 'import secrets,string; alphabet=string.ascii_letters+string.digits; print("".join(secrets.choice(alphabet) for _ in range(24)) + "aA1!", end="")' > "$pwfile"
        chown root:root "$pwfile"
        chmod 0400 "$pwfile"
      fi

      python3 - "$pwfile" "${adminEmail}" "${stepsFile}" <<'PY'
      import pathlib, sys
      pw = pathlib.Path(sys.argv[1]).read_text().strip()
      email, dest = sys.argv[2], sys.argv[3]
      text = f"""FirstInstance:
        InstanceName: Example
        DefaultLanguage: en
        Org:
          Name: Example
          Human:
            UserName: {email}
            FirstName: Admin
            LastName: User
            DisplayName: Administrator
            PreferredLanguage: en
            Email:
              Address: {email}
              Verified: true
            Password: "{pw}"
            PasswordChangeRequired: true
      """
      pathlib.Path(dest).write_text(text)
      PY
      chown zitadel:zitadel ${stepsFile}
      chmod 0400 ${stepsFile}
    '';
  };

  systemd.services.zitadel = {
    after = [
      "postgresql.target"
      "zitadel-bootstrap.service"
    ];
    requires = [
      "postgresql.target"
      "zitadel-bootstrap.service"
    ];
    serviceConfig = {
      # The socket in /run/postgresql is owned by the postgres group.
      SupplementaryGroups = [ "postgres" ];
      WorkingDirectory = stateDir;
      RestartSec = "3s";
    };
  };

  security.acme.acceptTerms = true;
  security.acme.defaults.email = adminEmail;

  networking.firewall.allowedTCPPorts = [ 80 443 ];

  services.nginx = {
    enable = true;
    virtualHosts.${domain} = {
      forceSSL = true;
      enableACME = true;
      http2 = true;
      locations."/" = {
        extraConfig = ''
          grpc_pass grpc://127.0.0.1:${toString listenPort};
          grpc_set_header Host $host;
          grpc_set_header X-Forwarded-Proto https;
          grpc_buffer_size 16k;
        '';
      };
    };
  };
}
```

`enableACME` needs port 80 free so the HTTP challenge can answer. If you already have a certificate, drop `enableACME` and set `sslCertificate` and `sslCertificateKey` on that virtual host instead.

Add the file to `configuration.nix`:

```nix
imports = [ ./services/zitadel.nix ];
```

Point DNS for `auth.example.com` at this machine, then:

```bash
sudo nixos-rebuild switch
```

## First login

The login name is the email address in the configuration, `admin@example.com`.

Zitadel appends `@<org>.<external-domain>` when `UserName` is a short name. `admin` in an org named `Example` on `auth.example.com` would sign in as `admin@example.auth.example.com`. Setting `UserName` to the email avoids that.

The generated password is only on the machine:

```bash
sudo cat /var/lib/zitadel/admin-password
```

Open `https://auth.example.com/ui/console`. Zitadel asks for a new password on that first login. The file under `/var/lib` is not updated when you change it.

## Check that it came up

```bash
systemctl status postgresql.service zitadel.service nginx.service
curl -fsS https://auth.example.com/debug/healthz
curl -fsS https://auth.example.com/debug/ready
curl -fsS https://auth.example.com/.well-known/openid-configuration
curl -fsS https://auth.example.com/saml/v2/metadata
```

`debug/healthz` returns `ok`. The OpenID discovery document's `issuer` is `https://auth.example.com`. The SAML metadata `entityID` uses that same host. A local `curl` to `127.0.0.1:8088` can succeed while the public name still fails. That means the proxy is not speaking h2c, or it is rewriting `Host`.

An nginx error `upstream sent too big header` is the gRPC buffer. `grpc_buffer_size 16k` in the example is there for that.

## Another proxy on port 443

nginx does not have to be the proxy. Whatever owns 443 has to do the same three things:

- send `auth.example.com` to `http://127.0.0.1:8088`
- use HTTP/2 cleartext on that connection
- keep `Host: auth.example.com` and set `X-Forwarded-Proto: https`

`ExternalPort` stays `443` and `ExternalSecure` stays `true`. Zitadel builds every issuer and redirect from those values. A public URL that includes another port will not match.

Omit the `services.nginx` block, the ACME settings, and the firewall ports in that case. Keep the PostgreSQL, Zitadel, and bootstrap configuration.

## After the first start

- Keep `/var/lib/zitadel/master.key`. Deleting it and letting the bootstrap unit write a new one makes the existing database unreadable.
- Keep the PostgreSQL major version. The cluster lives in `/var/lib/postgresql/17` when `package` is `pkgs.postgresql_17`.
- `postgresql.target` is the unit to wait for, not only `postgresql.service`. The role and the database are created by `postgresql-setup.service`, which is part of that target.
- Do not set `services.postgresql.authentication`. The module's default, `local all all peer`, is what the socket login uses. Replacing it with a TCP `trust` rule is what puts a passwordless database user on localhost.
