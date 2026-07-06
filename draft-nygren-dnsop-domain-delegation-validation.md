---
title: "Domain Control Validation for DNS Delegations"
docname: draft-nygren-dnsop-domain-delegation-validation-latest
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
stream: IETF
smart_quotes: no
pi: [toc, sortrefs, symrefs]
v: 3
abbrev: "Domain Control Validation for DNS Delegations"

workgroup: "Domain Name System Operations"

venue:
  group: "Domain Name System Operations"
  type: "Working Group"
  mail: "dnsop@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dnsop/"
  github: "enygren/draft-nygren-dnsop-domain-delegation-validation"
  latest: "https://enygren.github.io/draft-nygren-dnsop-domain-delegation-validation/draft-nygren-dnsop-domain-delegation-validation.html"

author:

 -
    ins: E. Nygren
    name: Erik Nygren
    organization: Akamai Technologies
    email: erik+ietf@nygren.org

 -
    ins: P. Thomassen
    name: Peter Thomassen
    organization: deSEC
    email: peter@desec.io

 -
    ins: S. Huque
    name: Shumon Huque
    organization: Salesforce
    email: shuque@gmail.com

normative:

informative:

    I-D.draft-ietf-dnsop-domain-verification-techniques:

    I-D.draft-ietf-deleg:

    SUBDOMAIN-TAKEOVER:
        title: "Subdomain takeovers"
        author:
          - ins: Mozilla
        target: https://developer.mozilla.org/en-US/docs/Web/Security/Subdomain_takeovers

    SITTING-DUCKS:
        title: "Don’t Let Your Domain Name Become a Sitting Duck"
        author:
          - name: Brian Krebs
        target: https://krebsonsecurity.com/2024/07/dont-let-your-domain-name-become-a-sitting-duck/

...

--- abstract

The techniques specified in {{I-D.draft-ietf-dnsop-domain-verification-techniques}} for using the DNS to verify ownership or control of a domain in the Domain Name System (DNS) rely on the domain already being properly delegated to a DNS authority. For the specific case where the Application Service Provider is providing authoritative DNS services, the existing approaches don't provide a way to bootstrap domain validation onto a new Authoritative DNS Application Service provider.

This specification proposes a mechanism for "Domain Control Validation" for cases where the User and DNS Administrator are validating control over a domain to an Authoritative DNS Application Service Provider and thus do not have the ability to add records within the domain. In this case, validation must be performed by the User taking actions in the DNS Registrar or Parent Zone that demonstrate control over the domain, such as by adding DNS `NS` records as validation records.

--- middle


# Introduction

Application Service Providers providing Authoritative DNS services need a way for domain owners (Users) to prove that they control a particular DNS domain before the Application Service Provider can allow them to provision the authoritative domain, thus granting a DNS Administrator access to modify the domain.

Security researchers have called out (see {{SITTING-DUCKS}}) that there is active exploitation of "lame delegations" ({{?RFC9499}}) where domain registrations remain active and delegate to an Authoritative DNS Application Service Provider, but where the User account at that Application Service Provider has lapsed or where they have "deleted" the domain but not removed the delegation. Without validation, an attacker may be able to provision that domain at the Application Service Provider and gain control over it. For new domains there is also a potential race condition during registration where the domain owner who registers (and delegates) a domain could be beat out by an attacker in provisioning the domain in the Authoritative DNS Application Service Provider.

While DNS `TXT` record validation as specified in {{I-D.draft-ietf-dnsop-domain-verification-techniques}} works for existing domains that the domain's DNS Administrator can add records to (such as for transferring to a different Authoritative DNS Application Service Provider), this does not work for domains that are newly provisioned or where the User and DNS Administrator are trying to re-establish control over.

For these cases, using DNS to demonstrate control over the domain must be performed through adding or modifying parent-side zone records, such as DNS `NS` records ({{?RFC1035}}).

This specification is also useful for performing validation during account recovery cases at Authoritative DNS Application Service Providers, where a User needs to validate that they have the control needed for a 2FA or password reset for a domain.

While non-DNS mechanisms are possible, such as using EPP authInfo ({{?RFC5731}}) or having the DNS Registrar add attributes to RDAP ({{?RFC9083}}), those are not described in this specification.


# Conventions and Definitions

{::boilerplate bcp14}

This specification uses the following terms defined in {{I-D.draft-ietf-dnsop-domain-verification-techniques}}: `Application Service Provider`, `DNS Administrator`, `Intermediary`, `User`, `Unique Token`, and `Validation Record`.

This specification uses the following terms defined in {{?RFC9499}} for general DNS Terminology: `Registrar`, `Registrant`, `Registry`, `Superordinate`/`Parent`, `Subordinate`/`Child`, and `Lame Delegation`.


# Purpose of Domain Control Validation for Delegation {#purpose}

Domain Control Validation for Delegation allows a User to demonstrate to an Application Service Provider providing Authoritative DNS services that the User's DNS Administrator should be given control over a Child Zone corresponding to a domain.

Because this challenge becomes publicly visible as soon as it is published into the DNS, the security properties rely on the causal relationship between the Application Service Provider generating a specific challenge and the challenge appearing in the DNS at a specified location.

For Domain Control Validation for Delegation, the DNS challenge must appear in a Parent Side record.

For the case where the domain is delegated from a delegation-centric zone operated by a DNS Registry, this addition needs to demonstrate that the User is also the DNS Registrant in control of the domain. The causal relationship in this case is:

~~~
   Application Service Provider for Authoritative DNS (challenge generation)
   -> User / Registrant for Child/Subordinate Domain
   -> Registrar
   -> Registry
   -> Parent/Superordinate DNS Zone update
   -> Application Service Provider for Authoritative DNS (for validation)
~~~

In the case where a domain is delegated from a parent that is not a delegation-centric zone in a registry, the Parent Zone DNS Administrator is making changes to demonstrate control over the zone, with a causal relationship of:

~~~
   Application Service Provider for Authoritative DNS (challenge generation)
   -> User / Registrant for Child Domain
   -> Parent Zone DNS Administrator
   -> Parent DNS Zone update
   -> Application Service Provider for Authoritative DNS (for validation)
~~~


# Threat Model {#threat-model}

The threat model here is an extension of that in  {{I-D.draft-ietf-dnsop-domain-verification-techniques}}, however this is the more narrow and specific case since the application (Authoritative DNS service) is known.

The specific threats we are concerned with include:

* T1. An attacker is able to exploit a pre-existing lame delegation to provision Authoritative DNS services at the target of the lame delegation without approval from the domain owner.
* T2. An attacker is able to exploit a race condition in the DNS registration and/or delegation process to gain control over the zone for a domain at an Authoritative DNS Application Service Provider. This might happen either before or after the domain has been delegated, but prior to the actual owner of the domain provisioning control.
* T3. An attacker might try to transfer a domain within an Authoritative DNS Application Service Provider from the rightful owner/operator's account to an account controlled by the attacker.
* T4. An attacker who can spoof DNS responses might be able to defeat validation, absent DNSSEC validation.

Since information in the DNS is public, the attacker should be assumed to have access to any validation records.

Threats NOT covered by this include:

* N1. Compromise of the Authoritative DNS Application Service Provider or compromise of the User's account there. This could also be used by an attacker to take over control of the domain.
* N2. A User retaining control over a domain at a DNS Application Service Provider when a new owner/Registrant for the domain delegates it.

Note that covered threat T3 and not-covered threat N2 are in-tension. When a domain transfers ownership at the parent without moving between Authoritative DNS Application Service Providers, it should be possible to transfer operational control to the new owner, but extreme care must be taken to authorize this transfer.

As threats T1 and T2 are mitigated by one-off validation, there is no need for validation records to persist.


# Recommendations {#recommendations}

Domain Control Validation during DNS delegation is implemented through DNS NS Record based validation as described below.

## NS Record based Validation {#ns-record}

The RECOMMENDED method of doing DNS-based domain control validation is to use a DNS `NS` record containing a Unique Token as the Validation Record within one of the parent-side zone NS record delegations. The Authoritative DNS Application Service Provider supplies an additional DNS authority name to use that contains this Unique Token.

The Unique Token SHOULD be encoded with Base32 encoding ({{!RFC4648, Section 6}}) or hexadecimal base16 encoding ({{!RFC4648, Section 8}}).

The QNAME is the domain being validated, and the RDATA MUST contain a nameserver name that contains a Unique Token provided by the Application Service Provider (constructed according to the properties described in {{I-D.draft-ietf-dnsop-domain-verification-techniques}}).

For example, in the parent-side zone:

~~~ dns
   $domain  IN NS   ns1.example.com.
   $domain  IN NS   ns2.example.com.
   $domain  IN NS   ns-<unique-token>.authdnsdv.example.net.
~~~

To validate these, the Application Service Provider queries for the parent-side NS records for the domain being validated.

Application Service Providers MUST validate that one of the parent-side DNS NS records is the DNS authority name they provided to that User for that specific domain and contains the matching Unique Token. The Application Service Provider MUST allow other NS records to also be present alongside the NS record being used for validation.

Note that using a recursive resolver to perform the validation may not be possible due to the need to distinguish child-side from parent-side NS records.

The validation NS record MUST be included in-addition to (rather than instead of) other NS records which will persist following the removal of the validation NS record.

Token metadata is not readily possible with this approach.

### Authoritative name servers for NS validation records

The Application Service Provider SHOULD provide the NS record authoritative server name for validation from a different TLD (top-level domain) from the domain being validated.

Because the validation authoritative server name would then be out-of-bailiwick relative to the domain being validated, it would not receive glue records in the parent zone's delegation response, requiring resolvers to perform a separate address resolution before it can be queried. This may make some resolver implementations less likely to select it during ordinary server selection, compared to already-glued in-bailiwick authorities, reducing (but not eliminating) the likelihood of this validation-only NS record being used for ongoing production query traffic.

The Application Service Provider MUST operate DNS service on IP addresses returned by the A/AAAA records for the authoritative server name, and those IPs must be treated as authoritative name servers for the zone.

Note that Registrars often have restrictions preventing in-bailiwick authorities from being referenced without first provisioning them, which is another reason for using a name from an unrelated TLD.

For example, one setup for validating `foo.example` might include:

~~~ dns
   foo.example. IN NS   ns1.dnsprovider.example.
   foo.example. IN NS   ns2.dnsprovider.example.
   foo.example. IN NS   ns-<unique-token>.authdnsdv.example.net.
   ns1.dnsprovider.example. IN AAAA 2001:db8:1::1
                            IN A    203.0.113.1
   ns2.dnsprovider.example. IN AAAA 2001:db8:2::2
                            IN A    192.0.2.5
   *.authdnsdv.example.net. IN AAAA 2001:db8:1::5
                            IN A    203.0.113.1
~~~

which allows the `ns*.dnsprovider.example` to be configured as authorities to be returned in Additional records within the `.example` domain.

If the example domain were to be within `.net` then authorities outside of `.net` would want to be used for validation purposes.


## DELEG SvcParam based Validation (future) {#deleg-svcparam}

Once DELEG ({{I-D.draft-ietf-deleg}}) is widely deployed, another option would be for a SvcParam to be registered for validation purposes. This would be defined in a future specification but would include the validation Unique Token along with the delegation:

~~~
foo.example.  DELEG include-delegparam=config2.example.net. validation=<unique-token>
~~~

## Removal and Cleanup

After validation is performed, the User SHOULD remove the validation record from the parent zone.

# Supporting Multiple Authoritative DNS Providers

Nothing in this specification prevents a zone from having multiple Authoritative DNS Providers, as the NS records may simultaneous include authoritative name servers from the multiple providers (possibly including ones used for validation).

# Security Considerations

Many of the Security Considerations are shared with {{I-D.draft-ietf-dnsop-domain-verification-techniques}} and are not repeated here. Additional considerations follow.


## DNS Spoofing and DNSSEC Validation

The parent side zone SHOULD be signed using DNSSEC {{?RFC9364}} to protect Validation Records against DNS spoofing attacks, including from on-path attackers.

Application Service Providers MUST use a trusted DNSSEC validating resolver to verify Validation Records they have requested to be deployed.

Application Service Providers MUST confirm that the NS validation records are present in the parent-side zone.


# Privacy Considerations

TODO - consider if any exist.

# IANA Considerations

This document has no IANA actions.


--- back


# Acknowledgments

Thank you to Peter Thomassen, ..., (add names here) for their feedback and suggestions on this document.

