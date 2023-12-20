[![ansible-lint](https://github.com/sscheib/ansible-role-generate_ssl_key_pairs/actions/workflows/ansible-lint.yml/badge.svg)](https://github.com/sscheib/ansible-role-generate_ssl_key_pairs/actions/workflows/ansible-lint.yml) [![Publish latest release to Ansible Galaxy](https://github.com/sscheib/ansible-role-generate_ssl_key_pairs/actions/workflows/ansible-galaxy.yml/badge.svg?branch=main)](https://github.com/sscheib/ansible-role-generate_ssl_key_pairs/actions/workflows/ansible-galaxy.yml)

public_key_infrastructure
=========

This role generates Root Certificate Authorities (CA) and Intermediate Certificate Authorities (CA). The Intermediate CA/s should be used to sign Certificate
Signing Requests (CSR).

A typical scenario would be to have a Root CA and multiple Intermediate CAs:
root.ca.example.com
  - intermediate1.ca.example.com
  - intermediate2.ca.example.com
  - intermediate3.ca.example.com

With this role you can create multiple Root CAs with each of them "having" multiple Intermediate CAs.

This role is slightly different from my other roles, as I assert that the variables are defined properly when including the tasks to create either the Root CAs or the
Intermediate CAs. Further, there are *no* defaults set for the CAs. Everything *has to be defined*. This is done on purpose to ensure that the role is understood before
"randomly" creating CAs. Of course, an extensive example is included in this README.

Requirements
------------

This role makes use of the [`community.crypto`](https://github.com/ansible-collections/community.crypto) which is defined in `collections/requirements.yml`.

Role Variables
--------------

| variable                                     | default                               | required | description                                                           |
| :---------------------------------           | :-----------------------------------  | :------- | :---------------------------------------------------------------------|
| `pki_root_certificate_authorities`           | unset                                 | true     | Definition of the Root CA/s and their optional Intermediate CA/s      |
| `pki_quiet_assert`                           | `true`                                | true     | Whether to quiet assert statements                                    |

## `pki_root_certificate_authorities` in detail

Essentially the variable is a list of Root CAs which all can optionally contain multiple Intermediate CAs. The Intermediate CAs are signed by the Root CA they are listed
within.

Imagine the variable like so:
```
- name: 'root-ca1.example.com'
  # lots of variables
  intermediates:
    - name: 'intermediate1.root-ca1.example.com'
      # lots of variables

    - name: 'intermediate2.root-ca1.example.com'
      # lots of variables

    - name: 'intermediate3.root-ca1.example.com'
      # lots of variables

- name: 'root-ca2.example.com'
  # lots of variables
  intermediates:
    - name: 'intermediate1.root-ca2.example.com'
      # lots of variables

    - name: 'intermediate2.root-ca2.example.com'
      # lots of variables

    - name: 'intermediate3.root-ca2.example.com'
      # lots of variables
```

With the above example `root-ca1.example.com` will sign the Intermediates `intermediate1.root-ca1.example.com`, `intermediate2.root-ca1.example.com` and
`intermediate3.root-ca1.example.com`.

`root-ca2.example.com` will sign the Intermediates `intermediate1.root-ca2.example.com`, `intermediate2.root-ca2.example.com`, `intermediate3.root-ca2.example.com`.

Why multiple Root CAs you ask? My use case is that I have multiple Root CA and Intermediate CA pairs for each use case *per sub-domain*. Yes, it's complicated.

For starting off I'd recommend to use **one Root CA** with **Intermediate CA**.


In below example you'll find references to the modules being used on specific options. I don't mean to replicate the excellent documentation of the `community.crypto`
collection, thereforce I rather ask you to look in the documentation for specific options.

Below you'll find an example with one Root CA and two Intermediate CAs:

```
pki_root_certificate_authorities:
  # the name is only used to identify each CA - it is not used otherwise
  - name: 'pki.example.com'

    # CA root directory + permissions of the directory
    root_dir_path: '/root/ansible_ca'
    root_dir_owner: 'root'
    root_dir_group: 'root'
    root_dir_mode: '0700'

    # CA private keys directory + permissions of the directory
    # This is whereere the private keys will be stored
    priv_key_dir_path: '/root/ansible_ca/private'
    priv_key_dir_owner: 'root'
    priv_key_dir_group: 'root'
    priv_key_dir_mode: '0700'

    # CA certificates (aka public keys) directory + permissions of the directory
    # This is where the public keys will be stored
    cert_dir_path: '/root/ansible_ca/certs'
    cert_dir_owner: 'root'
    cert_dir_group: 'root'
    cert_dir_mode: '0700'

    # CA Certificate Signing Request (CSR) certs directory + permissions of the directory
    # This is where the CSRs will be stored
    csr_dir_path: '/root/ansible_ca/csr'
    csr_dir_owner: 'root'
    csr_dir_group: 'root'
    csr_dir_mode: '0700'

    # CA Certificate Revocation List (CRL) certs directory + permissions of the directory
    # This is where the CRLs will be stored
    crl_dir_path: '/root/ansible_ca/crl'
    crl_dir_owner: 'root'
    crl_dir_group: 'root'
    crl_dir_mode: '0700'

    # CA private key
    #
    # This key will sign the intermediates
    # 
    
    # Path to the private key
    priv_key_path: '/root/ansible_ca/private/ca.key.pem'

    # Password of the private key
    priv_key_pass: !vault |
           $ANSIBLE_VAULT;1.1;AES256

    # Type of key to generate
    priv_key_type: 'RSA'

    # Length/Size of the Private Key
    priv_key_size: '4096'

    # Whether to force generation: THIS WILL OVERWRITE YOUR CURRENT KEY. USE WITH CAUTION!
    priv_key_force_generation: false   # see community.crypto.openssl_privatekey

    # permissions of the private key file
    priv_key_owner: 'root'
    priv_key_group: 'root'
    priv_key_mode: '0400'

    # Regeneration mode for the private key. If you manually fiddle around with the CA, I
    # encourage you to make use of 'fail', which will not regenerate the key if it differs
    # from the options defined.
    priv_key_regenerate: 'full_idempotence'  # see community.crypto.openssl_privatekey

    # Currently this *must* be 'auto' - there is no other value available (yet)
    priv_key_pass_cipher: 'auto'  # see community.crypto.openssl_privatekey

    # CA certificate (aka public key) 
    #
    # This key will be shared to clients that should trust the CA
    #

    # Permissions of the public key
    cert_path: '/root/ansible_ca/certs/ca.cert.pem'
    cert_owner: 'root'
    cert_group: 'root'
    cert_mode: '0400'

    # Certificate Signing Request (CSR)
    # CSR is generated with community.crypto.openssl_csr

    # Path to the CSR file
    csr_path: '/root/ansible_ca/csr/ca.csr.pem'

    # Common Name to use
    csr_common_name: 'pki.example.com - Ansible Certificate Authority'

    # Mail to use
    csr_email: 'steffen@scheib.me'

    # Organization (O) to use
    csr_org: 'Internal Infrastructure'

    # Country to use (C)
    csr_country: 'DE'

    # Organization Unit to use (OU)
    csr_org_unit: 'IT'

    # State or Province to use (ST)
    csr_state: 'BW'

    # Locality to use (L)
    csr_loc: 'Home'

    # Certificate Revocation List (CRL)
    # CRLs are generated using community.crypto.x509_crl
    #
    # This file holds revoked certificates and will be shared with
    # clients that need to check for revoked certificates.
    # Need to be regenerated regularly to be useful.

    # Whether to enable CRL
    crl_enable: true

    # Path and permissions to the CRL
    crl_path: '/root/ansible_ca/crl/ca.crl.pem'
    crl_owner: 'root'
    crl_group: 'root'
    crl_mode: '0400'

    # This is the expiry date of the CRL
    crl_next_update: '+180d'  # see community.crypto.x509_crl

    # Generation mode of the CRL file
    crl_generation_mode: 'update'   # see community.crypto.x509_crl module -> crl_mode

    # Whether to ignore timestamps for idempotence
    crl_ignore_timestamps: true  # see community.crypto.x509_crl

    # List of intermediates for the CA
    # The intermediates have basically the same options available, with one
    # exception: Chain Certificate Generation. Please see the first intermediate
    # for the options and description.
    intermediates:
      - name: 'office.pki.example.com'

        # CA chain certificate
        #

        # Whether to enable the creation of a chain certificate
        chain_certificate_enable: true


        # Chain certificate directory + permissions
        chain_certificate_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/certs'
        chain_certificate_dir_owner: 'root'
        chain_certificate_dir_group: 'root'
        chain_certificate_dir_mode: '0700'


        # Path to the chain certificate
        # The chain certificate is assembled from the root CA's public key and
        # the intermediate's public key
        chain_certificate_path: '/root/ansible_ca/intermediates/office.pki.example.com/certs/ca-chain.cert.pem'
        chain_certificate_owner: 'root'
        chain_certificate_group: 'root'
        chain_certificate_mode: '0400'

        # CA root directory
        root_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com'
        root_dir_owner: 'root'
        root_dir_group: 'root'
        root_dir_mode: '0700'

        # CA private keys directory
        priv_key_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/private'
        priv_key_dir_owner: 'root'
        priv_key_dir_group: 'root'
        priv_key_dir_mode: '0700'

        # CA certificates (aka public keys) directory
        cert_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/certs'
        cert_dir_owner: 'root'
        cert_dir_group: 'root'
        cert_dir_mode: '0700'

        # CA CSR certs directory
        csr_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/csr'
        csr_dir_owner: 'root'
        csr_dir_group: 'root'
        csr_dir_mode: '0700'

        # CA CRL certs directory
        crl_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/crl'
        crl_dir_owner: 'root'
        crl_dir_group: 'root'
        crl_dir_mode: '0700'

        # private key
        priv_key_path: '/root/ansible_ca/intermediates/office.pki.example.com/private/intermediate.key.pem'
        priv_key_pass: 'dasiugdasiugdaskigda213109udsab'
        priv_key_type: 'RSA'
        priv_key_size: '4096'
        priv_key_force_generation: false
        priv_key_owner: 'root'
        priv_key_group: 'root'
        priv_key_mode: '0400'
        priv_key_regenerate: 'full_idempotence'
        priv_key_common_name: 'office.pki.example.com - Ansible Certificate Authority'
        priv_key_pass_cipher: 'auto'

        # CA certificate (aka public key) 
        cert_path: '/root/ansible_ca/intermediates/office.pki.example.com/certs/intermediate.cert.pem'
        cert_owner: 'root'
        cert_group: 'root'
        cert_mode: '0400'

        # CSR
        csr_path: '/root/ansible_ca/csr/office.pki.example.com.csr.pem'
        csr_common_name: 'office.pki.example.com - Ansible Certificate Authority'
        csr_email: 'steffen@scheib.me'
        csr_org: 'Internal Infrastructure'
        csr_country: 'DE'
        csr_org_unit: 'IT'
        csr_state: 'BW'
        csr_loc: 'Office'

        # CRL
        crl_enable: true
        crl_path: '/root/ansible_ca/intermediates/office.pki.example.com/crl/intermediate.crl.pem'
        crl_owner: 'root'
        crl_group: 'root'
        crl_mode: '0400'
        crl_next_update: '+180d'  # this is the expiry date of the CRL
        crl_generation_mode: 'update'   # see community.crypto.x509_crl module -> crl_mode
        crl_ignore_timestamps: true  # see community.crypto.x509_crl module ignore_timestamps

      - name: 'home.pki.example.com'

        # CA chain certificate
        chain_certificate_enable: true

        # chain certificate directory
        chain_certificate_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/certs'
        chain_certificate_dir_owner: 'root'
        chain_certificate_dir_group: 'root'
        chain_certificate_dir_mode: '0700'

        # assembled from the root CA's public key and the intermediate's public key
        chain_certificate_path: '/root/ansible_ca/intermediates/home.pki.example.com/certs/ca-chain.cert.pem'
        chain_certificate_owner: 'root'
        chain_certificate_group: 'root'
        chain_certificate_mode: '0400'

        # CA root directory
        root_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com'
        root_dir_owner: 'root'
        root_dir_group: 'root'
        root_dir_mode: '0700'

        # CA private keys directory
        priv_key_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/private'
        priv_key_dir_owner: 'root'
        priv_key_dir_group: 'root'
        priv_key_dir_mode: '0700'

        # CA certificates (aka public keys) directory
        cert_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/certs'
        cert_dir_owner: 'root'
        cert_dir_group: 'root'
        cert_dir_mode: '0700'

        # CA CSR certs directory
        csr_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/csr'
        csr_dir_owner: 'root'
        csr_dir_group: 'root'
        csr_dir_mode: '0700'

        # CA CRL certs directory
        crl_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/crl'
        crl_dir_owner: 'root'
        crl_dir_group: 'root'
        crl_dir_mode: '0700'

        # private key
        priv_key_path: '/root/ansible_ca/intermediates/home.pki.example.com/private/intermediate.key.pem'
        priv_key_pass: 'dasiugdasiugdaskigda213109udsab'
        priv_key_type: 'RSA'
        priv_key_size: '4096'
        priv_key_force_generation: false
        priv_key_owner: 'root'
        priv_key_group: 'root'
        priv_key_mode: '0400'
        priv_key_regenerate: 'full_idempotence'
        priv_key_common_name: 'home.pki.example.com - Ansible Certificate Authority'
        priv_key_pass_cipher: 'auto'

        # CA certificate (aka public key) 
        cert_path: '/root/ansible_ca/intermediates/home.pki.example.com/certs/intermediate.cert.pem'
        cert_owner: 'root'
        cert_group: 'root'
        cert_mode: '0400'

        # CSR
        csr_path: '/root/ansible_ca/csr/home.pki.example.com.csr.pem'
        csr_common_name: 'home.pki.example.com - Ansible Certificate Authority'
        csr_email: 'steffen@scheib.me'
        csr_org: 'Internal Infrastructure'
        csr_country: 'DE'
        csr_org_unit: 'IT'
        csr_state: 'BW'
        csr_loc: 'Home'

        # CRL
        crl_enable: true
        crl_path: '/root/ansible_ca/intermediates/home.pki.example.com/crl/intermediate.crl.pem'
        crl_owner: 'root'
        crl_group: 'root'
        crl_mode: '0400'
        crl_next_update: '+180d'  # this is the expiry date of the CRL
        crl_generation_mode: 'update'   # see community.crypto.x509_crl module -> crl_mode
        crl_ignore_timestamps: true  # see community.crypto.x509_crl module ignore_timestamps
```

## Revoking certificates

Should you have the need to revoke one or more Intermediate certificates, this can be accomplished when setting `revoked: true` to the respective certificate like so:

```
[..]
vars:
  pki_root_certificate_authorities:
    - name: 'pki.example.com'
      # lots of other variables here
      intermediates:
        - name: 'office.pki.example.com'
          # lots of other variables here
          revoked: true
```

This, of course, only makes sense if you use CRLs, which I highly encourage you to do. Once a certificate is revoked, you need to create new CRLs, for both the Root CA
and the Intermediate CA.

## Viewing the generated certificates

If you'd like to view the certificates that have been generated, the `openssl` command on the PKI host can be used.
Please find below a few examples how to view each of the certificates.

### Private key

```
openssl rsa -in /root/ansible_ca/private/ca.key.pem -check
Enter pass phrase for /root/ansible_ca/private/ca.key.pem:
RSA key ok
writing RSA key
-----BEGIN RSA PRIVATE KEY-----
[..]
-----END RSA PRIVATE KEY-----
```

### Public key

```
openssl x509 -in /root/ansible_ca/certs/ca.cert.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            28:62:4d:4f:42:d2:01:a6:70:44:4c:81:a5:b4:20:b4:c8:23:ba:06
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = DE, ST = BW, L = Home, O = Internal Infrastructure, OU = IT, CN = pki.example.com - Ansible Certificate Authority, emailAddress = steffen@me.example.com
        Validity
            Not Before: Dec 14 21:46:21 2023 GMT
            Not After : Dec 11 21:46:21 2033 GMT
        Subject: C = DE, ST = BW, L = Home, O = Internal Infrastructure, OU = IT, CN = pki.example.com - Ansible Certificate Authority, emailAddress = steffen@me.example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
                Modulus:
                  [..]
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                DNS:pki.example.com - Ansible Certificate Authority
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier: 
                [..]
    Signature Algorithm: sha256WithRSAEncryption
    [..]
```

### CSR

```
openssl req -text -noout -verify -in /root/ansible_ca/csr/ca.csr.pem 
verify OK
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C = DE, ST = BW, L = Home, O = Internal Infrastructure, OU = IT, CN = pki.example.com - Ansible Certificate Authority, emailAddress = steffen@me.example.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (4096 bit)
                Modulus:
                  [..]
                Exponent: 65537 (0x10001)
        Attributes:
        Requested Extensions:
            X509v3 Subject Alternative Name: 
                DNS:pki.example.com - Ansible Certificate Authority
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
      [..]
```

### CRL

```
openssl crl -inform PEM -text -noout -in /root/ansible_ca/crl/ca.crl.pem 
Certificate Revocation List (CRL):
        Version 2 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = DE, ST = BW, L = Home, O = Internal Infrastructure, OU = IT, CN = pki.example.com - Ansible Certificate Authority, emailAddress = steffen@me.example.com
        Last Update: Dec 14 21:51:56 2023 GMT
        Next Update: Jun 11 21:51:56 2024 GMT
No Revoked Certificates.
    Signature Algorithm: sha256WithRSAEncryption
      [..]
```

## Verifying the generated certificates

If you'd like to manually check that the Intermediate certificates are correctly signed by the Root CA, we can again use the `openssl` command for that and simply run it on
the PKI host we used.

### Verifying Intermediate CA public key against a Root CA public key

To verify an Intermediate CA against a Root CA, we need the following command:

```
openssl verify -verbose -CAfile /root/ansible_ca/certs/ca.cert.pem /root/ansible_ca/intermediates/home.pki.example.com/certs/intermediate.cert.pem 
/root/ansible_ca/intermediates/home.pki.example.com/certs/intermediate.cert.pem: OK
```

### Verifying a generated certificate against Intermediate CA and Root CA
 
Verifying a certificate signed by an Intermediate CA means that `openssl` needs to know about the complete certificate chain. Root CA + Intermediate CA + the certificate to
actually validate.

With `openssl` it is not possible to use a generated chain certificate (so basically `cat ca.cert.pem intermediate.cert.pem > ca-chain.cert.pem`) without *explicitly trusting*
the intermediate certificate 'by accident'. Please see [this](https://mail.python.org/pipermail/cryptography-dev/2016-August/000676.html) mail posting which explains in great
detail on why *not* to use a chain certificate to verify a certificate using `openssl`.

So, what's the correct way then? `openssl` provides a command line switch: [`-untrusted`](https://www.openssl.org/docs/man3.0/man1/openssl-verify.html)

Below you'll find a short excerpt of the man page:

> A file or URI of untrusted certificates to use for chain building. This option can be specified more than once to load certificates from multiple sources.

Taking this into account, the command should look like the following:

```
openssl verify -CAfile /root/new_ca/certs/ca.cert.pem -untrusted /root/new_ca/intermediate/dev.pki.int.scheib.me/certs/intermediate.cert.pem /root/new_ca/intermediate/dev.pki.int.scheib.me/certs/openwrt.dev.int.scheib.me.cert.pem
```

If you have multiple Intermediate CAs, the order of the `-untrusted` switches does *not* matter.


### Verifying if a certificate is revoked

Checking for a revoked certificate is actually a *little* more complex. `openssl` needs to know about the Root CA, the Intermediate CA, the Root CRL and the Intermediate CRL.
Lastly, we'll need the certificate to check, of course. Luckily, `openssl` provides a way to provide more than one CRL file: [`-CRLfile`](https://www.openssl.org/docs/man3.0/man1/openssl-verify.html).

Below you'll find another short excerpt of the man page:

> The file or URI should contain one or more CRLs in PEM or DER format. This option can be specified more than once to include CRLs from multiple sources.

So simply checking a certificate against a Root CA, is pretty easy:

```
openssl verify -crl_check -CAfile /root/ansible_ca/certs/ca.cert.pem -CRLfile /root/ansible_ca/crl/ca.crl.pem /root/ansible_ca/intermediates/home.pki.example.com/certs/intermediate.cert.pem 
/root/ansible_ca/intermediates/home.pki.example.com/certs/intermediate.cert.pem: OK
```

With an Intermediate CA inbetween, it gets - unfortunately - a little more complicated. 

Basically `openssl` needs the following: Root CA public key + Intermediate CA public key + Root CA CRL + Intermediate CA CRL + certificate to check

This can look something like this (using an example from my own environment):

```
openssl verify -crl_check_all -CAfile /root/new_ca/certs/ca.cert.pem -untrusted /root/new_ca/intermediate/dev.pki.int.scheib.me/certs/intermediate.cert.pem -CRLfile /root/new_ca/crl/ca.crl.pem -CRLfile /root/new_ca/intermediate/dev.pki.int.scheib.me/crl/intermediate.crl.pem /root/new_ca/intermediate/dev.pki.int.scheib.me/certs/openwrt.dev.int.scheib.me.cert.pem 
/root/new_ca/intermediate/dev.pki.int.scheib.me/certs/openwrt.dev.int.scheib.me.cert.pem: OK
```

The order of the `-CRLfile` or `-untrusted` arguments does not matter.

Dependencies
------------

This role makes use of the collection [`community.crypto`](https://github.com/ansible-collections/community.crypto) and the collection [`ansible.posix`](https://github.com/ansible-collections/ansible.posix) which are both specified in `collections/requirements.yml`.

Example Playbook
----------------

```
---
- hosts: 'all'
  gather_facts: false
  roles:
    - 'sscheib.public_key_infrastructure'
  vars:
    pki_root_certificate_authorities:
      - name: 'pki.example.com'
        root_dir_path: '/root/ansible_ca'
        root_dir_owner: 'root'
        root_dir_group: 'root'
        root_dir_mode: '0700'
    
        # CA private keys directory + permissions of the directory
        # This is whereere the private keys will be stored
        priv_key_dir_path: '/root/ansible_ca/private'
        priv_key_dir_owner: 'root'
        priv_key_dir_group: 'root'
        priv_key_dir_mode: '0700'
    
        # CA certificates (aka public keys) directory + permissions of the directory
        # This is where the public keys will be stored
        cert_dir_path: '/root/ansible_ca/certs'
        cert_dir_owner: 'root'
        cert_dir_group: 'root'
        cert_dir_mode: '0700'
    
        # CA Certificate Signing Request (CSR) certs directory + permissions of the directory
        # This is where the CSRs will be stored
        csr_dir_path: '/root/ansible_ca/csr'
        csr_dir_owner: 'root'
        csr_dir_group: 'root'
        csr_dir_mode: '0700'
    
        # CA Certificate Revocation List (CRL) certs directory + permissions of the directory
        # This is where the CRLs will be stored
        crl_dir_path: '/root/ansible_ca/crl'
        crl_dir_owner: 'root'
        crl_dir_group: 'root'
        crl_dir_mode: '0700'
    
        # CA private key
        #
        # This key will sign the intermediates
        # 
        
        # Path to the private key
        priv_key_path: '/root/ansible_ca/private/ca.key.pem'
    
        # Password of the private key
        priv_key_pass: !vault |
               $ANSIBLE_VAULT;1.1;AES256
    
        # Type of key to generate
        priv_key_type: 'RSA'
    
        # Length/Size of the Private Key
        priv_key_size: '4096'
    
        # Whether to force generation: THIS WILL OVERWRITE YOUR CURRENT KEY. USE WITH CAUTION!
        priv_key_force_generation: false   # see community.crypto.openssl_privatekey
    
        # permissions of the private key file
        priv_key_owner: 'root'
        priv_key_group: 'root'
        priv_key_mode: '0400'
    
        # Regeneration mode for the private key. If you manually fiddle around with the CA, I
        # encourage you to make use of 'fail', which will not regenerate the key if it differs
        # from the options defined.
        priv_key_regenerate: 'full_idempotence'  # see community.crypto.openssl_privatekey
    
        # Currently this *must* be 'auto' - there is no other value available (yet)
        priv_key_pass_cipher: 'auto'  # see community.crypto.openssl_privatekey
    
        # CA certificate (aka public key) 
        #
        # This key will be shared to clients that should trust the CA
        #
    
        # Permissions of the public key
        cert_path: '/root/ansible_ca/certs/ca.cert.pem'
        cert_owner: 'root'
        cert_group: 'root'
        cert_mode: '0400'
    
        # Certificate Signing Request (CSR)
        # CSR is generated with community.crypto.openssl_csr
    
        # Path to the CSR file
        csr_path: '/root/ansible_ca/csr/ca.csr.pem'
    
        # Common Name to use
        csr_common_name: 'pki.example.com - Ansible Certificate Authority'
    
        # Mail to use
        csr_email: 'steffen@scheib.me'
    
        # Organization (O) to use
        csr_org: 'Internal Infrastructure'
    
        # Country to use (C)
        csr_country: 'DE'
    
        # Organization Unit to use (OU)
        csr_org_unit: 'IT'
    
        # State or Province to use (ST)
        csr_state: 'BW'
    
        # Locality to use (L)
        csr_loc: 'Home'
    
        # Certificate Revocation List (CRL)
        # CRLs are generated using community.crypto.x509_crl
        #
        # This file holds revoked certificates and will be shared with
        # clients that need to check for revoked certificates.
        # Need to be regenerated regularly to be useful.
    
        # Whether to enable CRL
        crl_enable: true
    
        # Path and permissions to the CRL
        crl_path: '/root/ansible_ca/crl/ca.crl.pem'
        crl_owner: 'root'
        crl_group: 'root'
        crl_mode: '0400'
    
        # This is the expiry date of the CRL
        crl_next_update: '+180d'  # see community.crypto.x509_crl
    
        # Generation mode of the CRL file
        crl_generation_mode: 'update'   # see community.crypto.x509_crl module -> crl_mode
    
        # Whether to ignore timestamps for idempotence
        crl_ignore_timestamps: true  # see community.crypto.x509_crl
    
        # List of intermediates for the CA
        # The intermediates have basically the same options available, with one
        # exception: Chain Certificate Generation. Please see the first intermediate
        # for the options and description.
        intermediates:
          - name: 'office.pki.example.com'
    
            # CA chain certificate
            #
    
            # Whether to enable the creation of a chain certificate
            chain_certificate_enable: true
    
    
            # Chain certificate directory + permissions
            chain_certificate_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/certs'
            chain_certificate_dir_owner: 'root'
            chain_certificate_dir_group: 'root'
            chain_certificate_dir_mode: '0700'
    
    
            # Path to the chain certificate
            # The chain certificate is assembled from the root CA's public key and
            # the intermediate's public key
            chain_certificate_path: '/root/ansible_ca/intermediates/office.pki.example.com/certs/ca-chain.cert.pem'
            chain_certificate_owner: 'root'
            chain_certificate_group: 'root'
            chain_certificate_mode: '0400'
    
            # CA root directory
            root_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com'
            root_dir_owner: 'root'
            root_dir_group: 'root'
            root_dir_mode: '0700'
    
            # CA private keys directory
            priv_key_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/private'
            priv_key_dir_owner: 'root'
            priv_key_dir_group: 'root'
            priv_key_dir_mode: '0700'
    
            # CA certificates (aka public keys) directory
            cert_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/certs'
            cert_dir_owner: 'root'
            cert_dir_group: 'root'
            cert_dir_mode: '0700'
    
            # CA CSR certs directory
            csr_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/csr'
            csr_dir_owner: 'root'
            csr_dir_group: 'root'
            csr_dir_mode: '0700'
    
            # CA CRL certs directory
            crl_dir_path: '/root/ansible_ca/intermediates/office.pki.example.com/crl'
            crl_dir_owner: 'root'
            crl_dir_group: 'root'
            crl_dir_mode: '0700'
    
            # private key
            priv_key_path: '/root/ansible_ca/intermediates/office.pki.example.com/private/intermediate.key.pem'
            priv_key_pass: 'dasiugdasiugdaskigda213109udsab'
            priv_key_type: 'RSA'
            priv_key_size: '4096'
            priv_key_force_generation: false
            priv_key_owner: 'root'
            priv_key_group: 'root'
            priv_key_mode: '0400'
            priv_key_regenerate: 'full_idempotence'
            priv_key_common_name: 'office.pki.example.com - Ansible Certificate Authority'
            priv_key_pass_cipher: 'auto'
    
            # CA certificate (aka public key) 
            cert_path: '/root/ansible_ca/intermediates/office.pki.example.com/certs/intermediate.cert.pem'
            cert_owner: 'root'
            cert_group: 'root'
            cert_mode: '0400'
    
            # CSR
            csr_path: '/root/ansible_ca/csr/office.pki.example.com.csr.pem'
            csr_common_name: 'office.pki.example.com - Ansible Certificate Authority'
            csr_email: 'steffen@scheib.me'
            csr_org: 'Internal Infrastructure'
            csr_country: 'DE'
            csr_org_unit: 'IT'
            csr_state: 'BW'
            csr_loc: 'Office'
    
            # CRL
            crl_enable: true
            crl_path: '/root/ansible_ca/intermediates/office.pki.example.com/crl/intermediate.crl.pem'
            crl_owner: 'root'
            crl_group: 'root'
            crl_mode: '0400'
            crl_next_update: '+180d'  # this is the expiry date of the CRL
            crl_generation_mode: 'update'   # see community.crypto.x509_crl module -> crl_mode
            crl_ignore_timestamps: true  # see community.crypto.x509_crl module ignore_timestamps
    
          - name: 'home.pki.example.com'
    
            # CA chain certificate
            chain_certificate_enable: true
    
            # chain certificate directory
            chain_certificate_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/certs'
            chain_certificate_dir_owner: 'root'
            chain_certificate_dir_group: 'root'
            chain_certificate_dir_mode: '0700'
    
            # assembled from the root CA's public key and the intermediate's public key
            chain_certificate_path: '/root/ansible_ca/intermediates/home.pki.example.com/certs/ca-chain.cert.pem'
            chain_certificate_owner: 'root'
            chain_certificate_group: 'root'
            chain_certificate_mode: '0400'
    
            # CA root directory
            root_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com'
            root_dir_owner: 'root'
            root_dir_group: 'root'
            root_dir_mode: '0700'
    
            # CA private keys directory
            priv_key_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/private'
            priv_key_dir_owner: 'root'
            priv_key_dir_group: 'root'
            priv_key_dir_mode: '0700'
    
            # CA certificates (aka public keys) directory
            cert_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/certs'
            cert_dir_owner: 'root'
            cert_dir_group: 'root'
            cert_dir_mode: '0700'
    
            # CA CSR certs directory
            csr_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/csr'
            csr_dir_owner: 'root'
            csr_dir_group: 'root'
            csr_dir_mode: '0700'
    
            # CA CRL certs directory
            crl_dir_path: '/root/ansible_ca/intermediates/home.pki.example.com/crl'
            crl_dir_owner: 'root'
            crl_dir_group: 'root'
            crl_dir_mode: '0700'
    
            # private key
            priv_key_path: '/root/ansible_ca/intermediates/home.pki.example.com/private/intermediate.key.pem'
            priv_key_pass: 'dasiugdasiugdaskigda213109udsab'
            priv_key_type: 'RSA'
            priv_key_size: '4096'
            priv_key_force_generation: false
            priv_key_owner: 'root'
            priv_key_group: 'root'
            priv_key_mode: '0400'
            priv_key_regenerate: 'full_idempotence'
            priv_key_common_name: 'home.pki.example.com - Ansible Certificate Authority'
            priv_key_pass_cipher: 'auto'
    
            # CA certificate (aka public key) 
            cert_path: '/root/ansible_ca/intermediates/home.pki.example.com/certs/intermediate.cert.pem'
            cert_owner: 'root'
            cert_group: 'root'
            cert_mode: '0400'
    
            # CSR
            csr_path: '/root/ansible_ca/csr/home.pki.example.com.csr.pem'
            csr_common_name: 'home.pki.example.com - Ansible Certificate Authority'
            csr_email: 'steffen@scheib.me'
            csr_org: 'Internal Infrastructure'
            csr_country: 'DE'
            csr_org_unit: 'IT'
            csr_state: 'BW'
            csr_loc: 'Home'
    
            # CRL
            crl_enable: true
            crl_path: '/root/ansible_ca/intermediates/home.pki.example.com/crl/intermediate.crl.pem'
            crl_owner: 'root'
            crl_group: 'root'
            crl_mode: '0400'
            crl_next_update: '+180d'  # this is the expiry date of the CRL
            crl_generation_mode: 'update'   # see community.crypto.x509_crl module -> crl_mode
            crl_ignore_timestamps: true  # see community.crypto.x509_crl module ignore_timestamps
...
```


License
-------

GPL v2 or later
