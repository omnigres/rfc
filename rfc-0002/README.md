# Postgres Extensions Ecosystem Engineer (EEE) Role RFC

## Summary

This RFC defines the Extension Ecosystem Engineer role – a specialized
engineering position focused on making Postgres extensions discoverable,
adoptable, and operationally production-ready. This role is distinct from
extension engineering and is rather focused on getting extensions into the
hands of users in real-life scenarios and keeping them there. It is a densely
packed role.

As this is a Request For Comments, it is essential to note that it undergoes a
discussion process, and we welcome feedback and other positive contributions.

## Motivation

Having a technical capability to extend the database is insufficient,
to the point of risking being more of an escape hatch rather than a
well-reputed growth mechanism. We can illustrate this by highlighting that most
hosted database providers impose a strict vetting process on extension
availability, limiting their growth potential. This is well-intended because
Postgres extensions wield nearly unlimited power and can be a source of
disruption, data loss, or breaches. These are higher-risk events considering
that we’re dealing with systems of record.

Current database and software teams often lack dedicated expertise to address
challenges in the extension ecosystem. This results in:

* Extensions remaining underutilized despite solving real problems
* Repeated tooling development across organizations
* Operational failures

Effectively, extension adoption is bottlenecked by operational complexity,
uncertainty and elevated risk profile.

This also suggests that the Postgres extension ecosystem is still in its
infancy and requires further development.

While this RFC is essential to Omnigres for accurately identifying the role as
well as finding and hiring the people for it, it is also intended as a focal
point to spell out this ecosystem’s challenges so that the broader community
can have a better overview and pointer for action plans.

The goal of making it an RFC is to make sure the role is shaped by those
participating in the ecosystem and, ultimately, those taking up the role, giving
clarity and agency to them to deliver the necessary outcomes and have a
positive impact. 

## Role Definition

### Primary Responsibilities

* Removing technical and social roadblocks to extension adoption and the growth of their use
* Building tools that facilitate the adoption and growth goals
* Working on individual extensions to ensure they are fit for delivery and  production use

## Required Technical Skills

**MUST have:**

* Deep Postgres administration knowledge (operational impacts of DDL, knowledge of catalogs, etc.) 
* Strong SQL and PL/pgSQL proficiency
* Command-line tooling development experience, scripting languages (Python, etc.) fluency

**SHOULD have:**

* Package management systems experience
* Database backup/restore operational knowledge
* High-availability deployment experience

**GREAT to have:**

* C, C++, Rust experience (at least comprehension for reading the source code)
* Understanding of extension architecture and internals
* Postgres core development experience

# Scope and Boundaries

### **In Scope**

* Extension use advocacy and interaction with customers using or considering their use
* Extension upgrade maintenance, tooling, and processes
* Extension build harnessing, packaging, distribution, and respective infrastructure
* Contributing to developing new ways of delivering extension to Postgres to mitigate current known and to-be-discovered issues
* Compatibility testing and validation
* Extension quality testing
* Operational tooling for extension management
* Documentation and best practices for extension adoption
* Contributing to core Postgres where high impact is possible
* Establishing adoption and operational metrics
* Collaborating in establishing extension development best practices and guidelines

### **Out of Scope**

* Individual extension feature development
* Application-specific database schema design
* General database administration tasks not related to extensions

Some of these areas may need to be involved during work, but they are not core to the job.

## **Success Criteria**

### **Technical Outcomes**

* Omnigres extensions (and other relevant, identified extensions) are suitable for production use, upgrades, and compatibility with other extensions.
* Deploying extensions to the cloud and on-premise is covered by well-tested tooling, processes, and services.
* Comprehensive tooling for extension lifecycle management

### **Adoption Metrics**

* Increased extension adoption rates across the ecosystem
* Reduced time-to-production for new extension deployments
* Changes for extension-related support ticket volume (both increase and decrease indicate different adoption phases)
* Community contribution to extension ecosystem tooling

### **Operational Impact**

* Extension upgrades become routine operational procedures
* Multi-extension deployments achieve production reliability
* Extension-related outages become rare events
* Extension infrastructure scales with organizational growth

## **Organizational Integration**

The Extension Ecosystem Engineer role fits within:

* Database Infrastructure
* Platform Engineering
* Site Reliability Engineering
* Developer Tools and Productivity
* Tech Advocacy

**Works closely with:**

* Database Administrators and Site Reliability Engineers for production deployment processes, operational concerns and requirements
* Application Developers for extension usage patterns
* Extension Developers for testing and quality assurance

# Drawbacks

The very cross-functional nature of this role makes it a challenging, “unicorn”
position, as there will never be a high industry demand for it – it’s a
mission-driven role with limited (but not small) impact radius.

# Alternatives

Keeping the status quo and let the existing roles share the focus.

# Unresolved Questions

None yet.
