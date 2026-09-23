# Certificate on the Zitadel virtual host

nginx requests a certificate for `auth.example.com` with the HTTP challenge. Port 80 has to reach this machine. See the [README](README.md) for the login and the health checks.

Save as `/etc/nixos/services/zitadel.nix` and import it from `configuration.nix`.

```nix
{ pkgs, ... }:

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
      pathlib.Path(dest).write_text(f"""FirstInstance:
        InstanceName: Example
        Org:
          Name: Example
          Human:
            UserName: {email}
            FirstName: Admin
            LastName: User
            Email:
              Address: {email}
              Verified: true
            Password: "{pw}"
            PasswordChangeRequired: true
      """)
      PY
      chown zitadel:zitadel ${stepsFile}
      chmod 0400 ${stepsFile}
    '';
  };

  systemd.services.zitadel = {
    after = [ "postgresql.target" "zitadel-bootstrap.service" ];
    requires = [ "postgresql.target" "zitadel-bootstrap.service" ];
    serviceConfig.SupplementaryGroups = [ "postgres" ];
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
        '';
      };
    };
  };
}
```
