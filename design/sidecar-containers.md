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

## Syntax

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

### Syntax note

Docker Compose allows a second syntax for environment variables and uses the key `environment`.
For consistency, I elected to stick with what we use everywhere else in Azure Pipelines: `env`.

## Requirements

We support spinning up one or more sidecar containers when the job starts.
The job itself may be in a container as well, or it may be running on the host.
Tasks and scripts must be able to access the network services offered by the sidecar containers.
All the containers that Azure Pipelines knows about will be on a shared Docker network.
Also, we must be able to map in somewhat arbitrary bind mounts.

### Networking for container jobs

A container job can access the other containers by hostname.
Ports specified in the container are automatically exposed.
`redis-cli -h redis ping` should work, assuming there's a container whose resource name is `redis`.

### Networking for host jobs

For a host job, we can't rely on Docker DNS for name resolution.
Customers can specify host:container port maps and then use the host port:
If the portspec is `- "8080:80"`, `curl localhost:8080` should work.

If multiple agents run on a single host, hard mapping to ports can cause conflicts.
Instead, customers should use ephemeral port maps: `- 6379` which will cause the host to pick a random, unused port number.
We'll provide environment variables which let the host know where to access services.
`redis-cli -h localhost -p $(agent.services.redis.ports.6379) ping` would be expected to work.

### Storage

Of the many volume formats Docker Compose offers, we'll implement three:
- Bind mounts from host: `/opt/data:/var/lib/mysql`
- Randomly-named bind mounts: `/var/lib/mysql` - fewer use cases for this, but no reason to block it
- Named volumes: `datavolume:/var/lib/mysql` - good for sharing data across several containers as well as persisting data on a private agent

There is no need to implement the "long" syntax.

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
