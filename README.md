# vault-pki-examples
# Certificate Revocation with Vault

## How Certificate Revocation Works in Vault

- % vault PKI Engine: Vault tracks all issued certificates (if `no_store=false` on the PKI role) and can revoke any certificate by its serial number.
- Revocation Triggers CRL Update: When a certificate is revoked, % vault automatically regenerates the Certificate Revocation List (CRL) and, if configured, updates OCSP responses. This ensures that any system validating certificates against Vault’s CRL or OCSP endpoint will immediately see the revoked status.
- Revocation API: You can revoke a certificate by its serial number using the % vault CLI or API (or Ansible). <https://developer.hashicorp.com/vault/api-docs/secret/pki#revoke-certificate>

## Network considerations

- The client machine (the one holding the certificate for auth) does NOT need to connect directly to % vault to have its certificate revoked.
- However, the systems that validate the certificate (e.g., servers, domain controllers, or other endpoints that the client connects to) must be able to check the revocation status—typically by accessing the CRL (Certificate Revocation List) or OCSP (Online Certificate Status Protocol) endpoint published by Vault.

## Best Practices

- Ensure all validating servers can reach Vault’s CRL/OCSP endpoints.
- Automate CRL/OCSP distribution if servers are in isolated networks (e.g., mirror the CRL internally).
- Monitor CRL/OCSP freshness—stale lists can allow revoked certs to be accepted.
- Use short-lived certificates to minimize risk if revocation checks are delayed.

## Ansible task to revoke a certificate

```YAML
- name: Revoke a certificate in Vault
  community.hashi_vault.hashi_vault_write:
    url: "{{ vault_addr }}"
    token: "{{ vault_token }}"
    path: "pki/revoke"
    data:
      serial_number: "{{ cert_serial_number }}" #either serial_number or certificate (but not both) must be specified on requests to this endpoint
```


## Deeper Dive into CRL and OCSP

### What is CRL?

CRL is the Certificate Revocation List. It is a published (static) list, which needs to be republished when a certificate is revoked. It can become large and stale. The list is downloaded periodically to maintain freshness, and a presented certificate is compared to the list to determine validity.

### What is OCSP?

OCSP is the Online Certificate Status Protocol. 

### CRL vs OCSP Revocation

CRL is a simple model that relies on file syncs, and supports offline or isolated networks. For a relatively small number of cert revocations at lower frequency, it is simple and effective. 

OCSP is a fast check that occurs per cert. The responder must be online for the check.

Resources:
- [PKI CRL OCSP Tutorial, from this point](https://developer.hashicorp.com/vault/tutorials/pki/pki-unified-crl-ocsp-cross-cluster#configure-pki-secrets-engines)
- [Set Revocation Configuration API endpoint](https://developer.hashicorp.com/vault/api-docs/secret/pki#set-revocation-configuration)
- [% vault Agent for automatic rendering of updated PKI certificates](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template#certificates)
- [Revocation: CRL and OCSP](https://www.golinuxcloud.com/tutorial-pki-certificates-authority-ocsp/#revocation-crl-and-ocsp)

## Walkthrough

The unified CRL and OCSP, and cross cluster revocation features are opt-in.

For environments with % vault Performance Replication, the primary cluster is not aware of certificates issued from a secondary cluster, unless the PKI secrets engine mount point is configured with support for unified CRL and OCSP.

If you need further details about these parameters, refer to their API documentation:

|                                          |                                                                                      |                                                                                         |
|------------------------------------------|--------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| auto_rebuild                             | auto rebuilds CRL                                                                    | https://developer.hashicorp.com/vault/api-docs/secret/pki#auto_rebuild                  |
| cross_cluster_revocation (not HCP Vault) | Enables cross-cluster revocation request queues, for Performance Replica secondaries | https://developer.hashicorp.com/vault/api-docs/secret/pki#cross_cluster_revocation      |
| unified_crl                              | Enables unified CRL and OCSP building, to synchronize all revocations between clusters | https://developer.hashicorp.com/vault/api-docs/secret/pki#unified_crl                   |
| unified_crl_on_existing_paths            | Enables serving the unified CRL and OCSP on the existing, previously cluster-local paths (e.g., `/pki/crl` will now contain the unified CRL when enabled). | https://developer.hashicorp.com/vault/api-docs/secret/pki#unified_crl_on_existing_paths |

## Demonstrate simple CRL certificate revocation

1) Spin up HCP % vault cluster
2) Export VAULT_ADDR, VAULT_NAMESPACE, and VAULT_TOKEN to env vars.
3) Start [PKI CRL OCSP Tutorial, from this point](https://developer.hashicorp.com/vault/tutorials/pki/pki-unified-crl-ocsp-cross-cluster#configure-pki-secrets-engines). Skip the performance replication setup.

```sh
% vault write pki-int-both/config/crl \
    auto_rebuild=true \
    unified_crl=true \
    unified_crl_on_existing_paths=true \
    cross_cluster_revocation=true

% % vault write \
  pki-int-local/issue/local-example-dot-com \
  common_name="test.local.example.com" \
  ttl="1h" -format=json > test.local.example.com.json

% cat test.local.example.com.json
% cat test.local.example.com.json | jq -r '.data.serial_number' > test.local.example.com.serial.txt
% cat test.local.example.com.serial.txt
% cat test.local.example.com.json | jq -r '.data.certificate' > test.local.example.com.crt
% vault write \
  pki-int-cross/issue/cross-example-dot-com \
  common_name="test.cross.example.com" \
  ttl="1h" -format=json > test.cross.example.com.json

% cat test.cross.example.com.json | jq -r '.data.certificate' > test.cross.example.com.crt
% cat test.cross.example.com.json | jq -r '.data.serial_number' > test.cross.example.com.serial.txt
% vault write \
  pki-int-both/issue/both-example-dot-com \
  common_name="test.both.example.com" \
  ttl="1h" -format=json > test.both.example.com.json
% cat test.both.example.com.json | jq -r '.data.serial_number' > test.both.example.com.serial.txt
% cat test.both.example.com.json | jq -r '.data.certificate' > test.both.example.com.crt
```


```sh
% openssl ocsp \  
    -noverify \
    -no_nonce \
    -issuer pki-int-local.cert.pem \
    -cert test.local.example.com.crt \
    -url $VAULT_ADDR/v1/admin/pki-int-local/ocsp
test.local.example.com.crt: good
        This Update: Sep 16 21:07:55 %GMT
        Next Update: Sep 17 09:07:55 %GMT
```

## Revoke cert

```sh
% vault write pki-int-local/revoke \
      serial_number=$(cat test.local.example.com.serial.txt)
```

## Cert shows as revoked

```sh
% openssl ocsp \
    -noverify \
    -no_nonce \
    -issuer pki-int-local.cert.pem \
    -cert test.local.example.com.crt \
    -url $VAULT_ADDR/v1/admin/pki-int-local/ocsp
test.local.example.com.crt: revoked
        This Update: Sep 16 21:47:14 %GMT
        Next Update: Sep 17 09:47:14 %GMT
        Revocation Time: Sep 16 21:08:39 %GMT
```