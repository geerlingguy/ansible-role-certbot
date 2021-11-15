# Ansible Role: Certbot (for Let's Encrypt)

[![CI](https://github.com/geerlingguy/ansible-role-certbot/workflows/CI/badge.svg?event=push)](https://github.com/geerlingguy/ansible-role-certbot/actions?query=workflow%3ACI)

Installs and configures Certbot (for Let's Encrypt).

## Requirements

If installing from source, Git is required. You can install Git using the `geerlingguy.git` role.

Generally, installing from source (see section `Source Installation from Git`) leads to a better experience using Certbot and Let's Encrypt, especially if you're using an older OS release.

## Role Variables

    certbot_install_method: package

Controls how Certbot is installed. Available options are 'package', 'snap', and 'source'.

    certbot_auto_renew: true
    certbot_auto_renew_user: "{{ ansible_user | default(lookup('env', 'USER')) }}"
    certbot_auto_renew_hour: "3"
    certbot_auto_renew_minute: "30"
    certbot_auto_renew_options: "--quiet --no-self-upgrade"

By default, this role configures a cron job to run under the provided user account at the given hour and minute, every day. The defaults run `certbot renew` (or `certbot-auto renew`) via cron every day at 03:30:00 by the user you use in your Ansible playbook. It's preferred that you set a custom user/hour/minute so the renewal is during a low-traffic period and done by a non-root user account.

### Automatic Certificate Generation

Currently there are two built-in methods for generating new certificates using this role: `standalone` and `dns`. Other methods (e.g. using nginx or apache and a webroot) may be added in the future.

**For a complete example**: see the fully functional test playbook in [molecule/default/playbook-standalone-nginx-aws.yml](molecule/default/playbook-standalone-nginx-aws.yml).

    certbot_create_if_missing: false
    certbot_create_method: standalone

Set `certbot_create_if_missing` to `yes` or `True` to let this role generate certs. Set the method used for generating certs with the `certbot_create_method` variable—current allowed values include: `standalone` and `dns`.

    certbot_admin_email: email@example.com

The email address used to agree to Let's Encrypt's TOS and subscribe to cert-related notifications. This should be customized and set to an email address that you or your organization regularly monitors.

    certbot_certs: []
      # - email: janedoe@example.com
      #   domains:
      #     - example1.com
      #     - example2.com
      # - domains:
      #     - example3.com

A list of domains (and other data) for which certs should be generated. You can add an `email` key to any list item to override the `certbot_admin_email`.

    certbot_create_command: "{{ certbot_script }} certonly --standalone --noninteractive --agree-tos --email {{ cert_item.email | default(certbot_admin_email) }} -d {{ cert_item.domains | join(',') }}"

The `certbot_create_command` defines the command used to generate the cert.

#### Standalone Certificate Generation

    certbot_create_standalone_stop_services:
      - nginx

Services that should be stopped while `certbot` runs it's own standalone server on ports 80 and 443. If you're running Apache, set this to `apache2` (Ubuntu), or `httpd` (RHEL), or if you have Nginx on port 443 and something else on port 80 (e.g. Varnish, a Java app, or something else), add it to the list so it is stopped when the certificate is generated.

These services will only be stopped the first time a new cert is generated.

#### DNS Certificate Generation

To use DNS challenge method when creating certificates you must use: `certbot_create_method: dns`. You have the following parameters:    

    certbot_dns_plugin: *dns-plugins*

    certbot_dns_target_server: *target.IP*
    certbot_dns_target_server_port: *DNS.Port*
    certbot_dns_tsig_keyname: "*tsig_key*"
    certbot_dns_key_secret: "*key/api_secret*"
    certbot_dns_key_algorithm: "*tsig_key_algorithm*"

If you are using another plugin instead of RFC2136, like CloudFlare or DigitalOcean `certbot_dns_key_secret` will be used as **API Token** and you do not need other parameters.    

DNS TSIG keys can be generate using `dnssec-keygen -a HMAC-SHA256 -b 256 -n HOST certbot.` but futher details and configurations are not covered here. The same applies for configuring DNS (BIND), but sample configuration is shown using BIND/RFC2136:

``` 
key certbot. {
  algorithm hmac-sha256;
  secret "9arciOclzevu7rEvSww0cpiOu5aPo65NFMBkcEuad5U=";
};

zone "example.com" {
  type master;
  ...
  allow-update { key certbot. };
};

```

#### DNS Plugins

Currently there are three built-in methods for **dns-plugins**: `rfc2136`, `cloudflare` and `digitalocean`. You can review DNS plugins supported by Certbot [here](https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins). Other methods (e.g. using nginx or apache and a webroot) may be added in the future.    
If you want to use other credentials files using other plugins you have to set `certbot_dns_credentials_custom_file: <file.path>`.  

#### DNS Integration - Deploy Hook

When using DNS integration, you do not need to stop/start service after certificate is generated. You only need to restart/reload service. There is a nice feature that already assembles *fullchain.pem* and *privkey.pem* into one file for use directly on haproxy. You use can enable this behavior by adding 'haproxy' to `certbot_create_dns_deploy_hook_services` variable:  

    certbot_create_dns_deploy_hook_services:
      - haproxy

It works exactly the same as `certbot_create_standalone_stop_services`.   

Configuring haproxy will not be included, just the "hook" for certbot to make easier for configurating it as certbot *knowns* when a new certificate is issued/renewed.  

#### Testing Certificates - Staging Server

To use Lets Encrypt Staging CA (Testing Certificate) instead of "Production" CA, you can set `certbot_use_staging_server` to `true`. Defaults to `false`.  

#### Delete Certificates

To delete certificates prior generating them you must set `certbot_delete_certificate` to `true`. Defaults to `false`. If you want to fully remove certificates and not generate them anymore you must set `certbot_create_if_missing` to `false`.  

### Snap Installation

Beginning in December 2020, the Certbot maintainers decided to recommend installing Certbot from Snap rather than maintain scripts like `certbot-auto`.

Setting `certbot_install_method: snap` configures this role to install Certbot via Snap.

This install method is currently experimental and may or may not work across all Linux distributions.

### Source Installation from Git

You can install Certbot from it's Git source repository if desired with `certbot_install_method: source`. This might be useful in several cases, but especially when older distributions don't have Certbot packages available (e.g. CentOS < 7, Ubuntu < 16.10 and Debian < 8).

    certbot_repo: https://github.com/certbot/certbot.git
    certbot_version: master
    certbot_keep_updated: true

Certbot Git repository options. If installing from source, the configured `certbot_repo` is cloned, respecting the `certbot_version` setting. If `certbot_keep_updated` is set to `yes`, the repository is updated every time this role runs.

    certbot_dir: /opt/certbot

The directory inside which Certbot will be cloned.

### Wildcard Certificates

Let's Encrypt supports [generating wildcard certificates](https://community.letsencrypt.org/t/acme-v2-and-wildcard-certificate-support-is-live/55579), but the process for generating and using them is slightly more involved. See comments in [this pull request](https://github.com/geerlingguy/ansible-role-certbot/pull/60#issuecomment-423919284) for an example of how to use this role to maintain wildcard certs.

Michael Porter also has a walkthrough of [Creating A Let’s Encrypt Wildcard Cert With Ansible](https://www.michaelpporter.com/2018/09/creating-a-wildcard-cert-with-ansible/), specifically with Cloudflare.

## Dependencies

None.

## Example Playbook

    - hosts: servers
    
      vars:
        certbot_auto_renew_user: your_username_here
        certbot_auto_renew_minute: "20"
        certbot_auto_renew_hour: "5"
    
      roles:
        - geerlingguy.certbot

See other examples in the `tests/` directory.

### Manually creating certificates with certbot

_Note: You can have this role automatically generate certificates; see the "Automatic Certificate Generation" documentation above._

You can manually create certificates using the `certbot` (or `certbot-auto`) script (use `letsencrypt` on Ubuntu 16.04, or use `/opt/certbot/certbot-auto` if installing from source/Git. Here are some example commands to configure certificates with Certbot:

    # Automatically add certs for all Apache virtualhosts (use with caution!).
    certbot --apache

    # Generate certs, but don't modify Apache configuration (safer).
    certbot --apache certonly

If you want to fully automate the process of adding a new certificate, but don't want to use this role's built in functionality, you can do so using the command line options to register, accept the terms of service, and then generate a cert using the standalone server:

  1. Make sure any services listening on ports 80 and 443 (Apache, Nginx, Varnish, etc.) are stopped.
  2. Register with something like `certbot register --agree-tos --email [your-email@example.com]`
    - Note: You won't need to do this step in the future, when generating additional certs on the same server.
  3. Generate a cert for a domain whose DNS points to this server: `certbot certonly --noninteractive --standalone -d example.com -d www.example.com`
  4. Re-start whatever was listening on ports 80 and 443 before.
  5. Update your webserver's virtualhost TLS configuration to point at the new certificate (`fullchain.pem`) and private key (`privkey.pem`) Certbot just generated for the domain you passed in the `certbot` command.
  6. Reload or restart your webserver so it uses the new HTTPS virtualhost configuration.

### Certbot certificate auto-renewal

By default, this role adds a cron job that will renew all installed certificates once per day at the hour and minute of your choosing.

You can test the auto-renewal (without actually renewing the cert) with the command:

    /opt/certbot/certbot-auto renew --dry-run

See full documentation and options on the [Certbot website](https://certbot.eff.org/).

## License

MIT / BSD

## Author Information

This role was created in 2016 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
