[[_TOC_]]


# Status quo of the OpenStudyBuilder

The OpenStudyBuilder solution introduces a new approach for working with studies that once fully implemented will drive end-to-end consistency and more efficient processes - all the way from protocol development and CRF design - to creation of datasets, analysis, reporting, submission to health authorities and public disclosure of study information.

OpenStudyBuilder is a next generation end-to-end clinical data standards and study specification solution, enabling clinical study data solutions to use linked metadata for higher degree of automation, limiting manual document driven work processes, enabling a Digital Data Flow approach.

OpenStudyBuilder is the open source version of the internal StudyBuilder solution at Novo Nordisk. Not all titles or logos in the application are yet changed to be 'OpenStudyBuilder' - when the term 'StudyBuilder' is used, it is therefore a synonym for 'OpenStudyBuilder'. This will be changed in coming updates.

For further information on the OpenStudyBuilder solution, please refer to the [OpenStudyBuilder homepage](https://openstudybuilder.com).

# System Requirements

A Docker environment with at least 6GB of memory allocated is required.

The solution is tested on Ubuntu and Windows (WSL 2).
For alternative platforms, please refer to [Platform architecture notes](platform-architecture-notes).

The Docker environment can be either Docker Desktop or Docker Engine 
(community version), with Compose V2 integrated as a CLI plugin. 

OpenStudyBuilder has been tested on the following Docker environments:

- Windows 11 (WSL 2)
- Ubuntu 20.04 Focal - Docker Engine community version 23.0.1 

To see versions of the installed Docker engine, run: `docker version`

To check if the Docker Compose V2 is also included, run: `docker compose version`

Windows installation link: [Windows installation](https://docs.docker.com/desktop/install/windows-install/)

Ubuntu installation link: [Ubuntu installation](https://docs.docker.com/engine/install/ubuntu/)

To test your local docker installation run the following command
in a non administrator or root shell: `docker run hello-world`

If this is not working see this link for Ubuntu rootless configuration:
[Docker Rootless](https://docs.docker.com/engine/security/rootless/)

For low-end systems, the database container may fail for low-on-memory 
reasons. In that case, update the following values in `compose.yaml`
```
        NEO4J_server_memory_heap_initial__size: "1G"
        NEO4J_server_memory_heap_max__size: "1G"
        NEO4J_server_memory_pagecache_size: "500M"
```

On Windows installations the WSL engine can take up all system resources.
It is recommended to configure limits. Create a `.wslconfig` file in the user directory, typically `C:\Users\username\`.
Put the following content in the `.wslconfig` file, and change `memory` to a suitable value for the given system. Half the physical RAM is a good starting point.
```
[wsl2]
processors=2
memory=6GB
```
See [Advanced settings configuration in WSL](https://docs.microsoft.com/en-us/windows/wsl/wsl-config)
for all available options.

# Using the docker environment

The Docker Compose configuration is `compose.yaml` for the preview 
environment and building containers in pipelines.

The following services are part of this Docker Compose environment.

- _database_ (A Neo4j graph database container including initial data)
- _api_ (A FastAPI container hosting the clinical-mdr-api backend application)
- _consumerapi_ (A FastAPI container hosting API for additional integrations)
- _neodash_ (Container for NeoDash, holding dashboards for OpenStudyBuilder)
- _frontend_ (A Nginx container hosting Vue.js StudyBuilder UI application)
- _documentation_ (A Nginx container hosting Vue.js Study Builder documentation portal)

## Starting the services

To start up the services for a local evaluation environment, use:

```shell
docker compose up
```

If you add the `-d` option to the command, it will bring up the services to 
run in the background, detached from the terminal.

To validate that the environment is running, inspect the output of this command:

```shell
docker compose ps
```

The output should look like this:

```
NAME                          IMAGE                       COMMAND                  SERVICE         CREATED        STATUS                            PORTS
dockerbuild-api-1             htp42/openstudybuilder:api-latest             "pipenv run uvicorn"     api             About a minute ago   Up 48 seconds (healthy)   8000/tcp
dockerbuild-consumerapi-1     htp42/openstudybuilder:consumerapi-latest     "pipenv run uvicorn"     consumerapi     About a minute ago   Up 48 seconds (healthy)   8000/tcp
dockerbuild-database-1        htp42/openstudybuilder:database-latest        "tini -g -- /startup…"   database        About a minute ago   Up 58 seconds (healthy)   127.0.0.1:5001->7474/tcp, 127.0.0.1:5002->7687/tcp
dockerbuild-documentation-1   htp42/openstudybuilder:documentation-latest   "/docker-entrypoint.…"   documentation   About a minute ago   Up 59 seconds (healthy)   80/tcp, 5006/tcp
dockerbuild-frontend-1        htp42/openstudybuilder:frontend-latest        "/docker-entrypoint.…"   frontend        About a minute ago   Up 17 seconds (healthy)   127.0.0.1:5005->5005/tcp
dockerbuild-neodash-1         htp42/openstudybuilder:neodash-latest         "/docker-entrypoint.…"   neodash         About a minute ago   Up 48 seconds (healthy)   127.0.0.1:5007->5007/tcp
```


## Accessing the application

- StudyBuilder main application: <http://localhost:5005/>

- StudyBuilder documentation: <http://localhost:5005/doc/>

  It can also be accessed from main web application from the ? sign in top right corner.

- StudyBuilder API (backend application): <http://localhost:5005/api/docs>

- StudyBuilder Consumer API (backend application): <http://localhost:5005/consumer-api/docs/>

- Neo4j dashboard web client: <http://localhost:5005/neodash/>

- Neo4j database web client: <http://localhost:5001/browser/>

  The default username is `neo4j` and the default password is `changeme1234`


## Stopping the services

This command will stop the docker containers, but keep the contents of the 
database for the next start.

```shell
docker compose down --remove-orphans
```

## Cleaning up the Docker environment

To clean up the entire Docker environment use the following commands:

**Will delete volumes and cache NOT restricted for the OpenStudyBuilder components.**

```shell
docker compose down --remove-orphans --volumes
docker rmi $(docker images --filter=reference="*_database" -q) -f
docker rmi $(docker images --filter=reference="*_api" -q) -f
docker rmi $(docker images --filter=reference="*_ui" -q) -f
docker rmi $(docker images --filter=reference="*_docs" -q) -f
docker rmi $(docker images --filter=reference="*_sonarqube" -q) -f
docker volume prune # (Will delete all volumes not used, only needed if -v was not used on docker-compose down command)
docker builder prune --all # (Will clean all docker cache files)
```
