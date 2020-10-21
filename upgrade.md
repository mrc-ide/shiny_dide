## Upgrade path

This system is a prototype that's been caught used in production, so this is worse than it should be.

The images are automatically built by buildkite. Update system dependencies in the dockerfiles in twinkle at [`shiny/Dockerfile`](https://github.com/mrc-ide/twinkle/blob/master/shiny/Dockerfile). If you make major changes, increase the version number in the file [`version`](https://github.com/mrc-ide/twinkle/blob/master/version) as this is used to create the tag that we use for deploying the applications.

## Starting the upgrade

If it's a big upgrade and we want to be careful, we can do the whole upgrade on the staging server and confirm that things are up and running.  Alternatively bring the system up on a differnet machine and confirm that things work that way.

You definitely want to do this on a machine which is on the Imperial network as there is a huge amount of data to download.

First, do the core setup with:

```
./scripts/init
./scripts/pull_images
./scripts/vault_auth
./scripts/configure_apache
```

At this point you will have a `docker-compose.yml` file that you can edit to expose the ports manually; update the admin block to include:

```
services:
  admin:
    ports: ["3838:3838"]
```

Get the test app provisioned

```
./scripts/provision hello
```

And bring up the system with no workers:

```
docker-compose up -d --scale shiny=0
```

Now, test that you can connect to the staging server. While on the VPN go to `http://129.31.x.y:3838/staging/hello/` (adjusting the IP as necessary).

## Updating apps

Theoretically we can do

```
./scripts/provision_all
./scripts/sync_server
```

to update all apps. This will possibly work, but it's not guaranteed and will be a bit tedious to recover from. After a major upgrade I worked through the list of apps and categorised problems - some were due to missing libraries on the server due to the upstream docker image changing, some where deploy keys that had been removed.

```
./scripts/provision msc-idm-2019
./scripts/provision msc-idm-2020
```

## Taking down the old version

All the volumes need clearing out for this to work as we can't have the previous packages interfering (they likely not work if the major version of R has changed for example).

Bring down the old system with

```
docker-compose stop
docker container prune
```

Mount the old volume into an ubuntu image and make a copy of it for safekeeping

```
docker volume rm shiny_dide_applications_backup
docker volume create shiny_dide_applications_backup
docker run --rm -it \
  -v shiny_dide_applications:/prev:ro \
  -v shiny_dide_applications_backup:/backup \
  alpine \
  cp -Rp /prev/. /backup/
```

Remove old volumes

```
docker volume rm shiny_dide_applications shiny_dide_staging
```

## Apps that require special care and tending:

* `msc-idm-2019`: `./helpers/reload-msc-idm-2019`
* `msc-idm-2019`: `./helpers/reload-msc-idm-2020`
* `infectiousdiseasemodels-2019`: `./helpers/reload-infectiousdiseasemodels-2019`
