## DIDE shiny server

### To deploy

```
ssh shiny
```

Then set everything up

```
./scripts/init
./scripts/pull_images
./scripts/vault_auth
./scripts/provision_all
./scripts/sync_server
./scripts/configure_apache
```

Do the actual deployment with

```
docker-compose up -d --scale shiny=12
./scripts/register_workers 12
```

### To update just one application

Updating one application can be faster, especially at the moment as there are false positives with package updates.

```
./scripts/init
./scripts/provision <appname>
./scripts/sync_app <appname>
```

If the package library does not need updating then `./scripts/provision` accepts an argument `--update-source-only` which just updates the application sources.

If you want to try provisioning from scratch, then pass in `--preclean`.

Once run, your app will be available as `https://shiny.dide.imperial.ac.uk:9000/staging/<appname>/` on the VPN

### To add a new application


**First, set up your repository so it can be automatically provisioned**.  To do this, add a file `provision.yml` to your repository so that dependencies can be automatically added.  This should live in the directory that your application lives in (i.e., alongside `app.R` or `server.R`/`ui.R`) and that might not be the root of the repository.  There are two probable types here.

1. if your repository is just a shiny app you will need something like [this](https://github.com/arranhamlet/POLICI/blob/cb97ebad7264e6d77206382b826303b769171d5c/provision.yml) with a list of packages:

```yaml
packages:
  - package1
  - package2
```

if this is alphabetical that's easier to keep up to date.  All packages used anywhere in your application need to be included (i.e., everything used by `library`, `require`, `::`, `:::`, `loadNamespace`, `requireNamespace`).  Don't worry about dependencies of those packages though, they'll be installed automatically.

2. If your repository is actually a package and the package is at the root of the repository, then you might want to use

```
self: true
packages:
  - package1
  - package2
```

Which will install your package (and its dependencies).  The `packages` section is optional and installs any additional packages required (packages that your package only `Suggests` are not included by default).

**Modify _this_ repository so that we know about your application**.  First, fork this repository.  Edit the [`site.yml`](site.yml) to add your new application.  In most cases this will look like

```yaml
  <path-for-your-app>:
    type: github
    spec: <github-account>/<repo-name>
    admin: <username>
```

The keys are:

- `<path-for-your-app>` where the application will be hosted.  If you use `superhampster` then your application will be hosted at https://shiny.dide.imperial.ac.uk/superhampster - there is no need for any match between this and your repository name.
- `<github-account>/<repo-name>` is similar to what is used for `install_github` - where to find the repo.  This is `mrc-ide/typochallenge` for https://github.com/mrc-ide/typochallenge - but if your application lives at a subdirectory within your repo and you want a specific branch, tag, or commit you might want to use the full spec such as `mrc-ide/superhampster/inst/app@release` which would use the application within the directory `inst/app` on the `release` branch of the repository `mrc-ide/superhampster` (see `remotes::parse_github_repo_spec` to play with the parseing of these specs)
- `<username>`: a username that you'd like on the server.  This will be used by the (currently in development) administration interface.  This does not need to be your DIDE username.

Please be aware that your application may be redeployed at any time.  If you want to prevent changes that you make being reflected in the source, please provide a stable branch (e.g., `@release`, `@stable`) or pin to a particular commit with the SHA.

**Is your repository private**: If so please add

```yaml
    auth:
      type: deploy_key
```

to the yaml above.  As part of the process Rich will send you a public key to add to your repo (if the repo is hosted on `mrc-ide` he will do this directly).

**Does your application use other private repositories**: If so please add

```yaml
    auth:
      type: github_pat
```

Create a GitHub personal access token [here](https://github.com/settings/tokens/new) with the description matching your application named and with the **repo** scope, and get that to Rich.

**Do you want a private application**: If so please add

```yaml
    groups:
      - group1
      - group2
```

to your yaml.  You should add a group to the bottom of the file too if none of the other groups look like what you want.  Just list usernames that you'd like and we can sort out the rest later.

**Let Rich know**: Create a pull request with your changes and tag `@richfitz` in the request, and let Rich know on slack too.  Please [tick the box](https://help.github.com/articles/allowing-changes-to-a-pull-request-branch-created-from-a-fork/) that lets me edit your pull request as it makes things a bit easier.

Join the `#shiny-server` channel on slack so that you are notified of changes.

**What happens next?**: Rich will try and deploy your application and may request a few changes so that it deploys smoothly.

**This seems like a giant faff**: I am trying to get this a bit automated so that once things are up and running you'll be able to administer your applications yourself, redeploying changes, checking logs, etc.  If you have access to docker you should also be able to test things more easily locally to make it more likely that we know things will work out the box.

### How does this work

See [`twinkle`](https://github.com/mrc-ide/twinkle) for the details.

### Other information

There is a `#shiny-server` channel on slack for status updates/problems etc.

### Notes for administrators

#### Adding a new private repo:

```
ssh shiny
./shiny_dide/scripts/vault_auth
./shiny_dide/scripts/add_deploy_key <spec>
```

it will print instructions that can be either followed directly (for `mrc-ide` repos) or to send to the repo owner (for personal repos).

#### Adding a new user

```
ssh shiny
pwgen 10 1
./shiny_dide/scripts/set_password <user>
```

Then run

```
./shiny_dide/scripts/configure_apache
```

to write out the configuration, and force apache to reload it with

```
docker exec shiny_dide_apache_1 apachectl -k graceful
```

#### In lieu of a better staging setup

1. approve and merge the PR


```
ssh shiny
cd shiny_dide
./scripts/provision <newapp>
```

#### Pushing `twinkle` changes up to the server

On whatever machine holds the `twinkle` source tree

```
./scripts/build
./scripts/pull_images
```

#### Updating an app without reprovisioning

This is useful with packages that make use of non-CRAN repos as we still get false positives there

```
./scripts/provision --update-source-only <appname>
```

#### Update a container

For example, to update the `admin` container:

```
./scripts/pull_images
docker-compose up --no-deps -d admin
```

#### Missing system dependencies

If a package fails because of missing system dependencies, add it to the Dockerfile for the `shiny` container (currently https://github.com/mrc-ide/twinkle/blob/master/shiny/Dockerfile) and rebuild in `twinkle` by running `./scripts/build` - if you get an incomprehensible error message about Debian then try pulling the source image with `docker pull rocker/shiny`

Then redeploy the admin container as above and try redeploying as above.

Once that works, redeploy the worker containers.
