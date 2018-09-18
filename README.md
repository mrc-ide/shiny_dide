## DIDE shiny server

To deploy

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
