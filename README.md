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
docker compose up -d -scale shiny=12
./scripts/register_workers 12
```

### To add a new application

1. Work on a branch or a fork of this repository
2. Edit [`site.yml`](site.yml) to add your new application
3. Create a pull request and make sure to tag `@richfitz` and/or ping Rich on slack

### Other information

There is a `#shiny-server` channel on slack for status updates/problems etc
