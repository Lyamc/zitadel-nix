# Zitadel on NixOS

Run Zitadel as a NixOS service. PostgreSQL is on the same machine. nginx terminates TLS and forwards HTTP/2 cleartext to Zitadel. No containers.

Pick one configuration:

- [Certificate requested by the Zitadel virtual host](with-acme.md)
- [Wildcard certificate from a separate `acme.nix`](wildcard.md)

Both use `https://auth.example.com` and `admin@example.com`. Change those before use.

The service block is the same in both files. The module has no listen address, so Zitadel binds every interface. Do not open port 8088.

nginx must proxy with `grpc_pass`. A normal `proxy_pass` does not speak HTTP/2 cleartext, and the console will not load. `ExternalDomain` is `auth.example.com`, `ExternalPort` is `443`, and `ExternalSecure` is `true`. Those three values are the public URL.

The database login is the Unix socket at `/run/postgresql`. Peer authentication maps the `zitadel` user to the `zitadel` role. The role needs `LOGIN`, `CREATEDB`, and `CREATEROLE`.

The master key and the first administrator password are created under `/var/lib/zitadel` and are not put in the Nix store. Keep the master key. Replacing it makes the existing database unreadable. The password file is only used while the instance is first created.

Sign in as `admin@example.com`. A short username such as `admin` is stored as `admin@example.auth.example.com`.

```bash
sudo cat /var/lib/zitadel/admin-password
```

Open `https://auth.example.com/ui/console`. The first login asks for a new password.

```bash
curl -fsS https://auth.example.com/debug/healthz
curl -fsS https://auth.example.com/.well-known/openid-configuration
curl -fsS https://auth.example.com/saml/v2/metadata
```

`debug/healthz` returns `ok`. The OpenID `issuer` and the SAML `entityID` both use `https://auth.example.com`.
