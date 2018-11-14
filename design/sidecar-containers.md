# "Sidecar" or "multi-container" support

An emerging pattern is the need for a "sidecar container".
Run your build and tests in a container, and they need the support of one or more other containers to complete.
Concrete scenario: Django is a Python-based web framework.
They support 5 different database backends out of the box.
So, a complete run of their test suite needs to happen against Postgres, MySQL, Microsoft SQL Server, Oracle, and MariaDB.

This is one of the problems that [Docker Compose](https://docs.docker.com/compose/) exists to solve.
We could let the user specify a Docker Compose file directly, but Docker Compose is too general.
We don't need multiple networks, deployment constraints, replicas, and restart policies.
Instead, we embed a mini-config inspired by Docker Compose v3 but without the overly general parts.

This example assumes we introduce inline container resource definitions.
We would also support our existing, separate-resource syntax.

```yaml
jobs:
- job: MyJob
  pool:
  container: 
    image: python:latest
    name: python-builder

## Multi-container support here
  services:
    redis:
      image: redis:alpine
      ports:
      - "6379"
    postgres:
      image: postgres:9.4
      volumes:
      - db-data:/var/lib/postgresql/data
      env:
        FOO: bar
## End multi-container

  steps:
  - script: ...
    displayName: Run tests
```
We would spin up the sidecar containers and make sure they're networked together with the main `python-builder` container.

## Syntax note

Docker Compose allows a second syntax for environment variables and uses the key `environment`.
I elected to stick with the standards we already established everywhere else in Azure Pipelines for consistency.

## Industry considerations

CircleCI has something like this.
```yaml
version: 2
jobs:
  build:
    working_directory: ~/mern-starter
    # The primary container is an instance of the first image listed. The job's commands run in this container.
    docker:
      - image: circleci/node:4.8.2-jessie
    # The secondary container is an instance of the second listed image which is run in a common network where ports exposed on the primary container are available on localhost.
      - image: mongo:3.4.4-jessie
```

Drone.io's config file is ["a superset of Docker Compose"](http://docs.drone.io/getting-started/#configuration), so they get this for free.
```yaml
pipeline:
  build:
    image: golang
    commands:
      - go get
      - go build
      - go test

services:
  postgres:
    image: postgres:9.4.5
    environment:
      - POSTGRES_USER=myapp
```

## Other notes
In the future, we could support Docker Compose files directly.
Starting small allows us to constrain the capabilities and retain flexibility.
For simple scenarios, something inline in the pipeline is preferable.

The need for multiple containers starts to break into deployment scenarios as well.
While this is a pure build'n'test play, we assume the deployment side will also want to read like a Docker Compose file.
This maintains a common vocabulary and fewer distinct concepts.
