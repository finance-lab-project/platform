# Current Finance Lab roadmap

The source of truth has moved to [finance-lab-project/app](https://github.com/finance-lab-project/app):

- [Architecture and accepted decisions](https://github.com/finance-lab-project/app/blob/main/docs/architecture.md)
- [Delivery sequence and dependencies](https://github.com/finance-lab-project/app/blob/main/docs/roadmap.md)
- [Release 1 umbrella](https://github.com/finance-lab-project/app/issues/1)

The current direction is a Java/Vaadin modular monolith without Spring, PostgreSQL Anchor Modeling and a custom ORM. Repositories are `app`, `orm`, `budget`, `portfolio` and `fire`; each has a private delivery board. Budget is the first implementation. Portfolio and FIRE remain design-only work for later releases.

The next design reviews are [Budget schema and constraints](https://github.com/finance-lab-project/budget/issues/1) and [runtime compatibility and version pinning](https://github.com/finance-lab-project/app/issues/2). Issue creation does not approve implementation, merge, deployment or spending.

The separate Finance Lab learning issue track is retired. Open learning issues were closed as not planned; existing completed history is retained. Current project boards track design and delivery work.

Earlier React/Spring, Go/BFF, local cluster and platform plans, including ADR-031 and the October reset documents, remain historical references. They do not override the linked architecture or reactivate old infrastructure work. Platform recovery safeguards still apply to any separately authorized work on those systems.
