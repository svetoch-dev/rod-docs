# rod

An attempt to build a cost-efficient, easy-to-manage, easy-to-extend infrastructure that supports multiple clouds.

Rod name derives from "род", a Russian word meaning lineage.

## Problem we are solving

Server software comes in many forms and serves many different purposes, but all server software needs the same basic foundation:

* Hardware to run on
* A way to understand what is happening now and what happened in the past
* A development lifecycle for delivering changes (`develop -> test -> deploy -> repeat`)

The combination of systems that fulfills these needs is called `infrastructure`.

Key aspects of good `infrastructure` are:

1. **Reliability**. The system should operate predictably and continue working even when parts of it fail. This includes high availability, fault tolerance, backups, disaster recovery, and graceful failure.
2. **Observability**. At any point in time, you should be able to understand what is happening in the system and what has happened before.
3. **Scalability**. The infrastructure should support growth without requiring a redesign every few months. This includes scaling traffic, data, teams, and the number of services.
4. **Simplicity**. Simple infrastructure is easier to understand, operate, debug, and hand over to new team members. Complexity is one of the biggest long-term costs in IT.
5. **Maintainability**. Good infrastructure should be easy to change safely. Clear structure, automation, documentation, standard patterns, and low operational burden are essential.
6. **Security**. Security should be built in from the start, not added later. Access control, network isolation, secret management, patching, auditing, and secure defaults are all critical.

Because infrastructure `needs` are fundamentally the same across most server software, we can create an `infrastructure` template that:
* **includes the key properties of good infrastructure**
* **can be reused for any kind of server software**

## This project is for

1. People who need infrastructure to test their idea, but do not want to spend endless hours managing or redesigning it when they grow from 10 daily active users (DAU) to 500,000.
2. People that need cost-efficient infrastructure
3. People that want to move fast without introducing technical debt

## State of the project

its still in a WIP(work in progress) state

### Cloud support

- [x] GCP
- [ ] Yandex Cloud
- [ ] Alibaba Cloud
- [ ] AWS
- [ ] Hetzner

