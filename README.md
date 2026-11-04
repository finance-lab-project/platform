# Finance Lab legacy platform

The current project home is [finance-lab-project/app](https://github.com/finance-lab-project/app).

- [Architecture and decisions](https://github.com/finance-lab-project/app/blob/main/docs/architecture.md)
- [Release roadmap](https://github.com/finance-lab-project/app/blob/main/docs/roadmap.md)
- [Release 1 delivery issue](https://github.com/finance-lab-project/app/issues/1)

Finance Lab is being redesigned as a Java/Vaadin modular monolith without Spring, with PostgreSQL Anchor Modeling and a custom ORM. Budget is implemented first; Portfolio and FIRE remain planned modules. The first release is a private Russian-language, RUB-only application on one Yandex Cloud VM.

This repository preserves platform experiments and governance. Earlier React/Spring, Go/BFF, Lima/K3s and GitOps roadmaps are historical and are not prerequisites for this release. The separate learning issue track is retired.

Follow [AGENTS.md](AGENTS.md) and individually approved issue scopes. These documents do not authorize infrastructure changes or deployment.
