## Jenkins Controller Deployment Guide

The following describes how to build, configure, back up, and restore a Jenkins controller (server) running inside Docker. Follow the sections in order when standing up a fresh environment, or jump to the relevant tasks for maintenance operations.

### Table of Contents
- [References](#references)
- [Initial Deployment](#initial-deployment)
- [Docker Volumes Backup](#docker-volumes-backup)
- [Restore Docker Volumes from Backup](#restore-docker-volumes-from-backup)
- [Updating a Running Jenkins Server Instance](#updating-a-running-jenkins-server-instance)

---

### References
- [Jenkins in Docker](https://www.jenkins.io/doc/book/installing/docker/)
- [Pre-installing Plugins](https://github.com/jenkinsci/plugin-installation-manager-tool/)
- [Configuration as Code](https://github.com/jenkinsci/configuration-as-code-plugin/tree/master)
- [Configuration as Code Plugin](https://plugins.jenkins.io/configuration-as-code/)
- [CASC Example](https://github.com/Praqma/praqma-jenkins-casc/blob/master/casc_configs/jenkins.yaml)
- [CASC Reload Endpoint](https://github.com/jenkinsci/configuration-as-code-plugin/blob/master/docs/features/configurationReload.md)
- [Backup/Restore Docker Volumes](https://docs.docker.com/engine/storage/volumes/#back-up-restore-or-migrate-data-volumes)

---

### Initial Deployment
1. Run the Docker-in-Docker (DinD) so the controller can reach the Docker daemon

  ```sh
  docker run \
    --name docker \
    --restart=on-failure \
    --detach \
    --privileged \
    --network jenkins \
    --network-alias docker \
    --env DOCKER_TLS_CERTDIR=/certs \
    --volume jenkins-docker-certs:/certs/client \
    --publish 2376:2376 \
    docker:dind
  ```

2. (Optional) Build the official jenkins-controller image

  ```sh
  docker build -t jenkins_controller:lts-jdk17 .
  ```

3. Update the required credentials entry inside [casc_configs/credentials.yaml](casc_configs/credentials.yaml) before starting the controller

4. Start the Jenkins controller container

  > **Note:** Replace `<PASSWORD>` and `<CASC_RELOAD_TOKEN>` with secure values before running the below command

  ```sh
  docker run \
    --name jenkins-controller \
    --restart=on-failure \
    --detach \
    --network jenkins \
    --network-alias jenkins-controller \
    --env DOCKER_HOST=tcp://docker:2376 \
    --env DOCKER_CERT_PATH=/certs/client \
    --env DOCKER_TLS_VERIFY=1 \
    --env JENKINS_ADMIN_PASSWORD=<PASSWORD> \
    --env JENKINS_LOCATION_URL=http://localhost:8080/ \
    --env CASC_RELOAD_TOKEN=<CASC_RELOAD_TOKEN> \
    --env PLUGINS_FORCE_UPGRADE=true \
    --publish 8080:8080 \
    --publish 50000:50000 \
    --volume jenkins-data:/var/jenkins_home \
    --volume jenkins-docker-certs:/certs/client:ro \
    jenkins_controller:lts-jdk17
  ```

---

### Docker Volumes Backup
The jenkins-controller uses two named volumes:

| Volume name            | Container path         | Mode |
|------------------------|------------------------|------|
| jenkins-docker-certs   | /certs/client          | ro   |
| jenkins-data           | /var/jenkins_home      | z    |

1. Create a tarball containing both volumes

  ```sh
  docker run --rm --volumes-from jenkins-controller -v "$(pwd)":/backup busybox:stable-musl tar cvf /backup/docker_volumes.tar /var/jenkins_home /certs/client
  ```

2. (Optional) Inspect the tar archive contents

  ```sh
  docker run --rm -v "$(pwd)":/backup busybox:stable-musl tar tf /backup/docker_volumes.tar
  ```

---

### Restore Docker Volumes from Backup
1. Ensure the named volumes exist

  ```sh
  docker volume inspect jenkins-data >/dev/null 2>&1 || docker volume create jenkins-data

  docker volume inspect jenkins-docker-certs >/dev/null 2>&1 || docker volume create jenkins-docker-certs
  ```

2. (Optional) Purge corrupt or stale data before restoring 
  
  > **Note:** Make sure the Jenkins controller is stopped before purging `jenkins_home`
  
  ```sh
  docker run --rm -v jenkins-data:/var/jenkins_home busybox:stable-musl sh -c 'rm -rf /var/jenkins_home/*'

  docker run --rm -v jenkins-docker-certs:/certs/client busybox:stable-musl sh -c 'rm -rf /certs/client/*'
  ```

3. Restore volumes from the tar backup

  ```sh
  docker run --rm -v jenkins-data:/var/jenkins_home -v "$(pwd)":/backup busybox:stable-musl sh -c 'cd /var/jenkins_home && tar xvf /backup/docker_volumes.tar --strip-components=2 var/jenkins_home'

  docker run --rm -v jenkins-docker-certs:/certs/client -v "$(pwd)":/backup busybox:stable-musl sh -c 'cd /certs/client && tar xvf /backup/docker_volumes.tar --strip-components=2 certs/client'
  ```

---

### Updating a Running Jenkins Server Instance

#### Update Configuration as Code (CASC)
1. Copy the updated CASC files into the container

  ```sh
  docker cp casc_configs/. jenkins-controller:/var/jenkins_home/casc_configs/
  ```

2. Trigger a CASC reload via the HTTP endpoint

  > **Note:** Ensure `<CASC_RELOAD_TOKEN>` matches the value configured on the controller

  ```sh
  curl -X POST "http://localhost:8080/reload-configuration-as-code/?casc-reload-token=<CASC_RELOAD_TOKEN>"
  ```

---

#### Update the Dockerfile
1. Stop and remove the running Jenkins container

  ```sh
  docker rm -f jenkins-controller
  ```

2. Rebuild and redeploy the controller image following the steps in [Initial Deployment](#initial-deployment)

3. Apply the refreshed CASC configuration using the steps in [Update Configuration as Code (CASC)](#update-configuration-as-code-casc)

---

#### Restore Volumes from Backup
1. Stop the running controller

  ```sh
  docker stop jenkins-controller
  ```

2. Restore the volumes by following [Restore Docker Volumes from Backup](#restore-docker-volumes-from-backup)

3. Start the controller to consume the restored data.

  ```sh
  docker start jenkins-controller
  ```

---