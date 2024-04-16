---
{"dg-publish":true,"permalink":"/Frontend/Database/","tags":["software","archi","data"],"noteIcon":""}
---

> [!abstract] Database
> This is the persistent storage for the system.
> It stores:
> - User credentials
> - Anchor positioning data
> 
> For future works, this may hold more information such as archival of processed location data, or even a storage for the Wi-Fi credentials for automated credential updating with the various components.

This is a PostgreSQL database.

As the database does not require much interference, it is Dockerised together with `PgAdmin` in a `docker-compose.yml` file, together with the components of [[Frontend/Frontend\|Frontend]] that interact with it. The [file](https://github.com/S32-PAMS/PSQL_DB/blob/main/docker-compose.yml) is in the [repository](https://github.com/S32-PAMS/PSQL_DB/tree/main).

> [!attention]
> For your implementation of PAMS, be sure to change the credentials and personal/organisation information in the `docker-compose.yml` file.

## Creation

To create this component:

```bash
git clone https://github.com/S32-PAMS/PSQL_DB.git
```

Make sure to change the fields accordingly.

Go back to [[Frontend/Frontend\|Frontend]] if setup there isn't complete.

## Running the connection

To run the PSQL_DB connection:

```bash
cd PSQL_DB
docker compose build --no-cache
docker compose up
```

