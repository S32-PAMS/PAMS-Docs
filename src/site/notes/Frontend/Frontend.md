---
{"dg-publish":true,"permalink":"/frontend/frontend/","noteIcon":""}
---

> [!abstract] Frontend
> This component refers to the web application, which provides the user interface for PAMS.
> 
> It communicate user credentials and anchor details with the [[Frontend/Database\|Database]], and streams camera footage with the [[Frontend/Camera Server\|Camera Server]].

The frontend architecture looks like this:

![frontendarchi.png](/img/user/Attachments/frontend-devs/frontendarchi.png)


> [!note] NextJS Web app
> This component hosts the frontend. It is based on [NextJS](https://nextjs.org/) and [NodeJS](https://nodejs.org/en).

## Installation

Install [NextJS](https://nextjs.org/) and [NodeJS](https://nodejs.org/en) on the hosting device. You can do this with various package managers, depending on your operating system.

> [!important]
> For PAMS prototype, ensure:
> - NodeJS version `>= v20.0`
> - NextJS version `>= v14.0`

- [Download NodeJS](https://nodejs.org/en/download/)
- [Install NextJS](https://nextjs.org/docs/getting-started/installation)

## Launch Prototype in Dev mode

After installing the require libraries, follow the steps below to launch the web application in Dev mode.

```bash
# Clone into frontend repo
git clone https://github.com/S32-PAMS/PAMS-FrontEnd.git

# Enter 'frontend' folder
cd frontend

# Install all dependencies
npm install

# Launch in dev mode
npm run dev
```

The default port for the frontend is set to `3000`.

> [!note]
> For ease of debugging, this guide aims to set up the web app locally. For Dockerisation, the web app will be in the same `docker-compose.yml` file as the [[Frontend/Database\|Database]], so you can refer to the [PSQL_DB repository](https://github.com/S32-PAMS/PSQL_DB/blob/main/docker-compose.yml). The `webapp` image has been commented out for debugging here, if this guide is followed to start it locally.

> [!attention]
> Now, leave this aside, and we need to set up the [[Frontend/Database\|Database]] to get the web application to work, so go to [[Frontend/Database\|Database]]!

## Connection to [[Frontend/Database\|Database]]

This section assumes you have completed building the [[Frontend/Database\|Database]].

This is how to connect to the database from this web application, with the credentials that correspond to those defined in `docker-compose.yml` for [[Frontend/Database#Creation\|Database#Creation]].

In `PAMS-FrontEnd` repo, set local environment variables with this [file](https://github.com/S32-PAMS/PAMS-FrontEnd/blob/main/frontend/.env.local), dummy example below:

```localenv
NEXT_PUBLIC_DUMMY_EMAIL = haha@gmail.com
NEXT_PUBLIC_DUMMY_PASSWORD = 123
JWT_SECRET = HOHOHAHA
NEXTAUTH_SECRET = HOOOOO
NEXTAUTH_URL=http://localhost:3000

DB_USER=postgres
DB_HOST=localhost
DB_NAME=pams
DB_PASSWORD=pams
DB_PORT=5432

CAM_LINK=http://localhost:1989/camera1/stream.m3u8
```

For every functionality implemented as an [API](https://github.com/S32-PAMS/PAMS-FrontEnd/tree/main/frontend/pages/api) in the `PAMS-FrontEnd` repo, they should have a code section as follows:

```ts
import { Pool } from 'pg';

const pool = new Pool({
    user: process.env.DB_USER,
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    password: process.env.DB_PASSWORD,
    port: parseInt(process.env.DB_PORT || "5432", 10),
});
```

This way, the functionality of the frontend will connect to the [[Frontend/Database\|Database]].

## Linking to [[Frontend/Camera Server\|Camera Server]]

For how to run the camera server in a complete PAMS prototype implementation, see [[Guides/For Running Server#Run the Camera Servers\|For Running Server#Run the Camera Servers]].

## Adding your Map

> [!important]
> This assumes you have completed setting up the [[Frontend/Map\|Map]]. If not, go there now, before integrating it into the user interface for visualisation!

Drop your `GeoJSON` layout file into the `frontend/components/MyMap` folder. Edit the import in `MyMap4.tsx` file in the same folder.

```typescript
import manualjson from "./<your geojson filename>.json";
```

Add the `png` of your layout into the `frontend/public` folder and edit the filename in the MyMap4 Object. Change the bounds of the object according to the metered dimensions of your map.

```typescript
<MyMap4 imglink=".\<filename>.png" boundsx={0} boundsy={3}/>
```

Push your changes to the PAMS-Frontend folder. Rerun [[Frontend/Frontend#Launch Prototype in Dev mode\|#Launch Prototype in Dev mode]].