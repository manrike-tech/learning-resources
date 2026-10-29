# Exercises – LDAP Clients with SSSD

<!-- nav:start -->
> **Module 02 · Users, Groups, Permissions & LDAP** — lesson 8 of 8  
> Read first: [Theory – Using LDAP for Users and Groups](../../theory/02-users-groups-permissions/08-ldap-clients.md) · [Course lesson (commands)](../../course/02-Users-Groups-Permissions.md#lesson-8---ldap-clients-ch-29) · [Module exercises](README.md)
<!-- nav:end -->

> Lab layout: `deb1` (Ubuntu) runs an OpenLDAP **server**; `rhel1` is the **client**. Replace `192.168.56.10` with deb1's IP. Add `192.168.56.10 ldap.example.lab` to `/etc/hosts` on both VMs.

Difficulty: ★ warm-up · ★★ practice · ★★★ challenge

---

## Exercise 1 – Build a test LDAP server (deb1) ★★

1. Install `slapd` and `ldap-utils`. Reconfigure with domain `example.lab`, organization `Lab`, admin password `adminpw`.
2. Load OUs, a group and two POSIX users from LDIF.

```bash
sudo apt install -y slapd ldap-utils
sudo dpkg-reconfigure slapd      # Omit config? No · DNS domain: example.lab · Org: Lab · password: adminpw
```

Create `/root/base.ldif`:

```text
dn: ou=people,dc=example,dc=lab
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=example,dc=lab
objectClass: organizationalUnit
ou: groups

dn: cn=devops,ou=groups,dc=example,dc=lab
objectClass: posixGroup
cn: devops
gidNumber: 10000
memberUid: ana
memberUid: luis

dn: uid=ana,ou=people,dc=example,dc=lab
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ana
cn: Ana Garcia
sn: Garcia
uidNumber: 10001
gidNumber: 10000
homeDirectory: /home/ana
loginShell: /bin/bash

dn: uid=luis,ou=people,dc=example,dc=lab
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: luis
cn: Luis Perez
sn: Perez
uidNumber: 10002
gidNumber: 10000
homeDirectory: /home/luis
loginShell: /bin/bash
```

**Verify:** `ldapsearch -x -b dc=example,dc=lab "(objectClass=posixAccount)" uid` lists ana and luis.

<details>
<summary>Solution</summary>

```bash
ldapadd -x -D cn=admin,dc=example,dc=lab -w adminpw -f /root/base.ldif
ldappasswd -x -D cn=admin,dc=example,dc=lab -w adminpw -s 'AnaPass1' uid=ana,ou=people,dc=example,dc=lab
ldappasswd -x -D cn=admin,dc=example,dc=lab -w adminpw -s 'LuisPass1' uid=luis,ou=people,dc=example,dc=lab
sudo ufw allow 389/tcp 2>/dev/null
```

</details>

---

## Exercise 2 – Query LDAP from the client ★

On `rhel1`, install the OpenLDAP client tools and:

1. list all entries' DNs under the base,
2. show only `uidNumber` and `homeDirectory` of `ana`,
3. list the members of `devops`.

<details>
<summary>Solution</summary>

```bash
dnf install -y openldap-clients
ldapsearch -x -H ldap://ldap.example.lab -b dc=example,dc=lab dn
ldapsearch -x -H ldap://ldap.example.lab -b dc=example,dc=lab "(uid=ana)" uidNumber homeDirectory
ldapsearch -x -H ldap://ldap.example.lab -b ou=groups,dc=example,dc=lab "(cn=devops)" memberUid
```

</details>

---

## Exercise 3 – Configure SSSD (rhel1) ★★

Configure `rhel1` so that LDAP users resolve with `getent` and can log in, with home directories created automatically.

> This lab server has no TLS. SSSD refuses password authentication over plain LDAP unless you add `ldap_auth_disable_tls_never_use_in_production = true`. **Only** use that in this lab; Exercise 5 adds TLS.

**Verify:**
```bash
getent passwd ana
id luis
su - ana       # as a normal user, password AnaPass1 – home is created
```

<details>
<summary>Solution</summary>

```bash
dnf install -y sssd sssd-ldap oddjob-mkhomedir
cat > /etc/sssd/sssd.conf << 'EOF'
[sssd]
services = nss, pam
domains = example.lab

[domain/example.lab]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ldap.example.lab
ldap_search_base = dc=example,dc=lab
ldap_id_use_start_tls = false
ldap_auth_disable_tls_never_use_in_production = true
cache_credentials = true
EOF
chown root:root /etc/sssd/sssd.conf && chmod 600 /etc/sssd/sssd.conf
authselect select sssd with-mkhomedir --force
systemctl enable --now sssd oddjobd
```

</details>

---

## Exercise 4 – Troubleshooting drills ★★

Break the client in each way below, observe the symptom, and fix it:

1. `chmod 644 /etc/sssd/sssd.conf` and restart sssd.
2. Change the search base to `dc=example,dc=com`, restart, and run `getent passwd ana`.
3. On the server, change `ana`'s `loginShell` to `/bin/sh` (`ldapmodify`). Why doesn't the client see it immediately? Force it.

<details>
<summary>Solution</summary>

1. `systemctl status sssd` / `journalctl -u sssd` → permissions error; `chmod 600`.
2. `getent` returns nothing; `ldapsearch` with the right base works → client config problem. Fix base, `systemctl restart sssd`, `sss_cache -E`.
3. SSSD caches entries. `sss_cache -E` (or `-u ana`) then `getent passwd ana`.

```bash
ldapmodify -x -D cn=admin,dc=example,dc=lab -w adminpw << 'EOF'
dn: uid=ana,ou=people,dc=example,dc=lab
changetype: modify
replace: loginShell
loginShell: /bin/sh
EOF
```

</details>

---

## Exercise 5 – Add TLS (StartTLS) ★★★

1. On `deb1`, create a lab CA and a certificate for `ldap.example.lab` (see [SSL Certificates](../../theory/08-security-vms-containers/04-ssl-certificates.md)) and configure slapd to use it (`olcTLSCertificateFile`, `olcTLSCertificateKeyFile`, `olcTLSCACertificateFile` via `ldapmodify -Y EXTERNAL -H ldapi:///`).
2. On `rhel1`, trust the CA and switch SSSD to `ldap_id_use_start_tls = true`, `ldap_tls_reqcert = demand`, removing the insecure option.

**Verify:** `ldapsearch -x -ZZ -H ldap://ldap.example.lab -b dc=example,dc=lab uid=ana` works on rhel1; `su - ana` works.

<details>
<summary>Solution (server side)</summary>

```bash
# deb1
openssl req -x509 -newkey rsa:4096 -nodes -keyout /etc/ldap/ca.key -out /etc/ldap/ca.crt -days 3650 -subj "/CN=Lab CA"
openssl req -newkey rsa:2048 -nodes -keyout /etc/ldap/ldap.key -out /tmp/ldap.csr -subj "/CN=ldap.example.lab"
openssl x509 -req -in /tmp/ldap.csr -CA /etc/ldap/ca.crt -CAkey /etc/ldap/ca.key -CAcreateserial \
  -out /etc/ldap/ldap.crt -days 825 -extfile <(echo "subjectAltName=DNS:ldap.example.lab")
chown openldap:openldap /etc/ldap/ldap.key; chmod 600 /etc/ldap/ldap.key
ldapmodify -Y EXTERNAL -H ldapi:/// << 'EOF'
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/ca.crt
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/ldap.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/ldap.key
EOF
```

```bash
# rhel1
scp deb1:/etc/ldap/ca.crt /etc/pki/ca-trust/source/anchors/lab-ca.crt && update-ca-trust
sed -i -e 's/^ldap_id_use_start_tls.*/ldap_id_use_start_tls = true/' \
       -e '/never_use_in_production/d' /etc/sssd/sssd.conf
echo 'ldap_tls_reqcert = demand' >> /etc/sssd/sssd.conf
systemctl restart sssd && sss_cache -E
```

</details>

---

<!-- footer:start -->
[← Exercises: User Resource Limits](07-resource-limits.md) · [Module exercises](README.md)
<!-- footer:end -->
