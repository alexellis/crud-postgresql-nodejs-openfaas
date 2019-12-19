# Guide to build a CRUD API with Postgresl and Node.js with OpenFaaS

In this guide you will learn how to create a CRUD API using Postgresql for storage, Node.js for application code, and OpenFaaS to automate Kubernetes. Since we are building on top of Kubernetes, we also get portability out of the box, scaling and self-healing infrastructure.

## Conceptual architecture

![Conceptual architecture](/images/crud.png)

We'll use the CRUD API for fleet management of a number of Raspberry Pis distributed around the globe. Postgresql will provide the backing datastore, OpenFaaS will provide scale-out compute upon Kubernetes, and our code will be written in Node.js using an OpenFaaS template and the pg library from npmjs.

## The guide

These are the component parts of the guide:

* [Kubernetes cluster](https://kubernetes.io/) - "Production-grade container orchestration"

    Created either on Civo using managed k3s (#KUBE100), using k3sup to provision k3s to one or more instances, or using any other Kubernetes service.

* [Postgresql](https://www.postgresql.org) - "The World's Most Advanced Database"

    We can install Postgresql using its helm chart, or we could use separate VM or managed service.

* [OpenFaaS](https://www.openfaas.com) - "Serverless Made Simple for Kubernetes"

    Once installed, OpenFaaS makes application development on Kubernetes simple. You can get an endpoint in production in a couple of minutes.

* [Docker](https://docker.com/) - container runtime

    You will also need Docker on your computer to build Docker images for use with OpenFaaS.

* [Node.js](https://nodejs.org) - sever-side JavaScript

    We'll build the CRUD API using the OpenFaaS `node12` template which uses Node.js. Node is a fast programming language which is well suited to I/O and networking.

### 1) Get your Kubernetes cluster

#### For KUBE100 users

If you're a part of the [#KUBE100 program](https://www.civo.com/blog/kube100-is-here), then create a new cluster in your Civo dashboard and configure your `kubectl` to point at the new cluster.

> For a full walk-through of Civo k3s you can see my blog post - [The World's First Managed k3s](https://blog.alexellis.io/the-worlds-first-managed-k3s/)

#### For everyone else

Alternatively, you can use any other Kubernetes cluster, or if you are on Civo already but not in #KUBE100, then create a new Small or Medium Instance, then use [k3sup ('ketchup')](https://k3sup.dev) to install k3s.

Before going any further, check that you are pointing at the correct cluster:

```sh
kubectl config get-contexts

kubectl get node -o wide
```

### 2) Install Postgresql

If you're a KUBE100 user, then you can add Postgresql as an application in your Civo dashboard.

For everyone else run the following.

* Install k3sup which can install Postgresql and OpenFaaS

```sh
curl -sLfS https://get.k3sup.dev | sudo sh
```

* Install Postgresql

```sh
k3sup app install postgresql
```

You should see output like this:

```
=======================================================================
= postgresql has been installed.                                      =
=======================================================================

PostgreSQL can be accessed via port 5432 on the following DNS name from within your cluster:

	postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

To connect to your database run the following command:

    kubectl run postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:11.6.0-debian-9-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql -U postgres -d postgres -p 5432

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/postgresql 5432:5432 &
	PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

# Find out more at: https://github.com/helm/charts/tree/master/stable/postgresql
```

Note the output carefully, it will show you the following, which you need in later steps:

* how to get your password
* how to connect with the Postgresql administrative CLI called `pgsql`

### 3) Install OpenFaaS

* Install the OpenFaaS CLI

```sh
curl -sLSf https://cli.openfaas.com | sudo sh
```

If you're a KUBE100 user, then you can add OpenFaaS as an application in your Civo dashboard.

For everyone else run the following.

* Install openfaas using k3sup

```sh
k3sup app install openfaas
```

Again, note the output because it will show you how to connect and fetch your password.

```
=======================================================================
= OpenFaaS has been installed.                                        =
=======================================================================

# Get the faas-cli
curl -SLsf https://cli.openfaas.com | sudo sh

# Forward the gateway to your machine
kubectl rollout status -n openfaas deploy/gateway
kubectl port-forward -n openfaas svc/gateway 8080:8080 &

# If basic auth is enabled, you can now log into your gateway:
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
echo -n $PASSWORD | faas-cli login --username admin --password-stdin

faas-cli store deploy figlet
faas-cli list

# For Raspberry Pi
faas-cli store list \
 --platform armhf

faas-cli store deploy figlet \
 --platform armhf

# Find out more at:
# https://github.com/openfaas/faas
```

### 4) Install Docker

If you do not have Docker on your local machine, then add it.

* [Docker Homepage](https://www.docker.com)

    Docker images provide a secure, repeatable, and portable way to build, and ship code between environments.

* Sign up for a [Docker Hub account](https://hub.docker.com)

    The Docker Hub is an easy way to share our Docker images between our laptop and our cluster.

    Run `docker login` and use your new username and password

    Now set your Docker username for use with OpenFaaS:

    ```sh
    export OPENFAAS_PREFIX="alexellis2"
    ```

### 5) Node.js

Now let's build the CRUD application that can store the status, uptime, and temperatures of any number of Raspberry Pis.

* Install Node.js 12 on your local computer

Get Node.js [here](https://nodejs.org/en/)

* Create a function using Node.js:

```sh
mkdir -p $HOME/crud/
cd $HOME/crud/

faas-cli new --lang node12 device-status

mv device-status.yml stack.yml
```

This creates three files for us:

```sh
├── device-status
│   ├── handler.js
│   └── package.json
└── stack.yml
```

* Add the `pq` npm module

Add the Postgresql library for Node.js (`pg`), we'll enable Pooling of connections, so that each HTTP request to our CRUD API does not create a new connection.

Here's what the Node.js library says about Pooling:

> The PostgreSQL server can only handle a limited number of clients at a time. Depending on the available memory of your PostgreSQL server you may even crash the server if you connect an unbounded number of clients.

> PostgreSQL can only process one query at a time on a single connected client in a first-in first-out manner. If your multi-tenant web application is using only a single connected client all queries among all simultaneous requests will be pipelined and executed serially, one after the other.

The good news is that we can access Pooling easily and you'll see that in action below.

```
cd $HOME/crud/device-status/
npm install --save pg
```

This step will update our `package.json` file.

### 6) Build the database schema

We'll have two tables, one will hold the devices and a key that they use to access the CRUD API, and the other will contain the status data.

```sql
-- Each device
CREATE TABLE device (
    device_id          INT GENERATED ALWAYS AS IDENTITY,
    device_key         text NOT NULL,
    device_name        text NOT NULL,
    device_desc        text NOT NULL,
    device_location    point NOT NULL,
    created_at         timestamp with time zone default now()
);

-- Set the primary key for device
ALTER TABLE device ADD CONSTRAINT device_id_key PRIMARY KEY(device_id);

-- Status of the device
CREATE TABLE device_status (
    status_id          INT GENERATED ALWAYS AS IDENTITY,
    device_id          integer NOT NULL references device(device_id),
    uptime             bigint NOT NULL,
    temperature_c      int NOT NULL,
    created_at         timestamp with time zone default now()
);

-- Set the primary key for device_status
ALTER TABLE device_status ADD CONSTRAINT device_status_key PRIMARY KEY(status_id);
```

Create both tables using a `pqsql` prompt, you took a note of this in the earlier step when you installed Postgresql with `k3sup`. The command given will run the `pgsql` CLI in a Kubernetes container.

Simply paste the text into the prompt.

* Provision a device

We'll provision our devices manually and use the CRUD API for the devices to call home.

Create an API key using bash:

```sh
export KEY=$(head -c 16 /dev/urandom | shasum | cut -d" " -f "1")
echo $KEY
```

```sql
INSERT INTO device (device_key, device_name, device_desc, device_location) values
('4dcf92826314c9c3308b643fa0e579b87f7afe37', 'k4s-1', 'OpenFaaS ARM build machine', POINT(35,-52.3));
```

At this point you'll get a new row, you'll need two parts for your device, its `device_id` which identifies it and its `device_key` which it will use to access the CRUD API.

```sql
SELECT device_id, device_key FROM device WHERE device_name = 'k4s-1';
```

Keep track of this for use with the CRUD API later:

```sql
 device_id |                device_key                
-----------+------------------------------------------
         1 | 4dcf92826314c9c3308b643fa0e579b87f7afe37
(1 row)
```

### 7) Build the CREATE operation

The C in CRUD stands for CREATE and we will use it to insert a row into the `device_status` database. The Raspberry Pi will run some code on a periodic basis using CRON, and access our API that way.

The connection pool code in the `pg` library needs several pieces of data to connect our function to the database.

* DB host (confidential)
* DB user (confidential)
* DB password (confidential)
* DB name (non-confidential)
* DB (non-confidential)

All confidential data will be stored in a Kubernetes secret, non-confidential data will be stored in `stack.yml` using environment variables.

* Create a secret named `db`

Remember that the instructions for getting the Postgresql password were given in the helm step.

```sh
export USER="postgres"
export PASS=""
export HOST="postgresql.default.svc.cluster.local"

kubectl create secret generic -n openfaas-fn db \
  --from-literal db-username="$USER" \
  --from-literal db-password="$PASS" \
  --from-literal db-host="$HOST"

secret/db created
```

* Attach the secret and environment variables to `stack.yml`

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080

functions:
  device-status:
    lang: node12
    handler: ./device-status
    image: alexellis2/device-status:latest
    environment:
      db_port: 5432
      db_name: postgres
    secrets:
      - db
```

* Edit `handler.js` and connect to the database

```js
"use strict"

const { Client } = require('pg')
const Pool = require('pg').Pool
const fs = require('fs')
var pool;

module.exports = async (event, context) => {
    if(!pool) {
        const poolConf = {
            user: fs.readFileSync("/var/openfaas/secrets/db-username", "utf-8"),
            host: fs.readFileSync("/var/openfaas/secrets/db-host", "utf-8"),
            database: process.env["db_name"],
            password: fs.readFileSync("/var/openfaas/secrets/db-password", "utf-8"),
            port: process.env["db_port"],
        };
        pool = new Pool(poolConf)
        await pool.connect()
    }

    let deviceKey = event.headers["x-device-key"]

    if(deviceKey && event.body.deviceID) {
        let deviceID = event.body.deviceID
        const { rows } = await pool.query("SELECT device_id, device_key FROM device WHERE device_id = $1 and device_key = $2", [deviceID,deviceKey]);
        if(rows.length) {
            await insertStatus(event, pool);
            return context.status(200).succeed({"status": "OK"});
        } else {
            return context.status(401).error({"status": "invalid authorization or device"});
        }
    }

    return context.status(200).succeed({"status": "No action"});
}

async function insertStatus(event, pool) {
    let id = event.body.deviceID;
    let uptime = event.body.uptime;
    let temperature = event.body.temperature;

    try {
        let res = await pool.query('INSERT INTO device_status (device_id, uptime, temperature_c) values($1, $2, $3)',
        [id, uptime, temperature]);
        console.log(res)
    } catch(e) {
        console.error(e)
    }
}
```

The code validates the header contains the correct device_key for the device ID specified in a request body.

Deploy the code with `faas up`

```
faas-cli up
```

* Now test it with `curl`:

```
curl 127.0.0.1:8080/function/device-status \
  --data '{"uptime": 2294567, "temperature": 38459}' \
  -H "X-Device-Key: 4dcf92826314c9c3308b643fa0e579b87f7afe37" \
  -H "X-Device-ID: 1" \
  -H "Content-Type: application/json"
```

Use `pqsql` to see if the row got inserted:

```sql
SELECT * from device_status;

 status_id | device_id | uptime  | temperature_c |          created_at           
-----------+-----------+---------+---------------+-------------------------------
         1 |         1 | 2294567 |         38459 | 2019-12-11 11:56:04.380975+00
(1 row)
```

If you ran into any errors then you can use a simple debugging approach:

* `faas-cli logs device-status` - this shows anything printed to stdout or stderr
* Add a log statement with `console.log("message")` and run `faas up` again

### 8) Build the RETRIEVE operation

Let's allow users or devices to select data when a valid device_key and device_id combination are given.

```js
"use strict"

const { Client } = require('pg')
const Pool = require('pg').Pool
const fs = require('fs')

const pool = initPool()

module.exports = async (event, context) => {  
    let client = await pool.connect()

    let deviceKey = event.headers["x-device-key"]
    let deviceID = event.headers["x-device-id"]

    if(deviceKey && deviceID) {
        const { rows } = await client.query("SELECT device_id, device_key FROM device WHERE device_id = $1 and device_key = $2", [deviceID, deviceKey]);
        if(rows.length) {

            if(event.method == "POST") {
                await insertStatus(deviceID, event, client);
                client.release()
                return context.status(200).succeed({"status": "OK"});
            } else if(event.method == "GET") {
                let rows = await getStatus(deviceID, event, client);
                client.release()
                return context.status(200).succeed({"status": "OK", "data": rows});
            }
            client.release()
            return context.status(405).fail({"status": "method not allowed"});
        } else {
            client.release()
            return context.status(401).fail({"status": "invalid authorization or device"});
        }
    }

    client.release()
    return context.status(200).succeed({"status": "No action"});
}

async function insertStatus(deviceID, event, client) {
    let uptime = event.body.uptime;
    let temperature = event.body.temperature;

    try {
        let res = await client.query('INSERT INTO device_status (device_id, uptime, temperature_c) values($1, $2, $3)',
        [deviceID, uptime, temperature]);
        console.log(res)
    } catch(e) {
        console.error(e)
    }
}

async function getStatus(deviceID, event, client) {
    let uptime = event.body.uptime;
    let temperature = event.body.temperature;

    let {rows} = await client.query('SELECT * FROM device_status WHERE device_id = $1',
    [deviceID]);
    return rows
}


function initPool() {
  return new Pool({
    user: fs.readFileSync("/var/openfaas/secrets/db-username", "utf-8"),
    host: fs.readFileSync("/var/openfaas/secrets/db-host", "utf-8"),
    database: process.env["db_name"],
    password: fs.readFileSync("/var/openfaas/secrets/db-password", "utf-8"),
    port: process.env["db_port"],
  });
 }
```

Deploy the code with `faas up`

```
faas-cli up
```

* Now test it with `curl`:

```
curl -s 127.0.0.1:8080/function/device-status \
  -H "X-Device-Key: 4dcf92826314c9c3308b643fa0e579b87f7afe37" \
  -H "X-Device-ID: 1" \
  -H "Content-Type: application/json"
```

If you have `jq` installed, it can also format the output, here's an example:

```
{
  "status": "OK",
  "data": [
    {
      "status_id": 2,
      "device_id": 1,
      "uptime": "2294567",
      "temperature_c": 38459,
      "created_at": "2019-12-11T11:56:04.380Z"
    }
  ]
}
```

The response body is in JSON and shows the underlying table schema. This can be "prettified" using an altered select statement, or by transforming each row in code to a new name.

### 8) Build client

We can now create a client for our CRUD API using any programming language, or even by using `curl` as above. Python is the best supported language for the Raspberry Pi ecosystem, so let's use that to create a simple HTTP client.

* Log into your [Raspberry Pi](https://www.raspberrypi.org) running [Raspbian](https://www.raspberrypi.org/downloads/raspbian/)

> Note you could also use a computer for this step, but must remove the temperature query since that will only work on the Raspberry Pi.

* Install Python3 and `pip`

```sh
sudo apt update -qy
sudo apt install -qy python3 python3-pip
```

* Add the Python `requests` module which can be used to make HTTP requests

```sh
sudo pip3 install requests
```

* Create a folder for the code

```sh
mkdir -p $HOME/client/
```

* Create a `client/requirements.txt` file

```
requests
```

* Create the `client/app.py`

```python
import requests
import os

ip = os.getenv("HOST_IP")

port = "31112"

deviceKey = os.getenv("DEVICE_KEY")
deviceID = os.getenv("DEVICE_ID")

uptimeSecs = 0
with open("/proc/uptime", "r") as f:
    data = f.read()
    uptimeSecs = int(data[:str.find(data,".")])

tempC = 0
with open("/sys/class/thermal/thermal_zone0/temp", "r") as f:
    data = f.read()
    tempC = int(data)

payload = {"temperature": tempC, "uptime": uptimeSecs}
headers = {"X-Device-Key": deviceKey, "X-Device-ID": deviceID, "Content-Type": "application/json"}

r = requests.post("http://{}:{}/function/device-status".format(ip, port), headers=headers, json=payload)

print(r.status_code)

```

* Create a script to run the client at `client/run_client.sh`

```
#!/bin/bash

export HOST_IP=91.211.152.145
export DEVICE_KEY=4dcf92826314c9c3308b643fa0e579b87f7afe37
export DEVICE_ID=1

cd /home/pi/client
python3 ./app.py
```

* Set up a schedule for the client using `cron`

```sh
chmod +x ./client/run_client.sh
sudo systemctl enable cron
sudo systemctl start cron

crontab -e
```

Enter the following to create a reading every 15 minutes.

```
*/15 * * * *    /home/pi/client/run_client.py
```

Test the script manually, then check the database table again.

```
/home/pi/client/run_client.py
```

Here are three consecutive runs I made using the command prompt:

```sql
postgres=# SELECT * from device_status;
 status_id | device_id | uptime  | temperature_c |          created_at           
-----------+-----------+---------+---------------+-------------------------------
         8 |         1 | 2297470 |         38459 | 2019-12-11 12:28:16.28936+00
         9 |         1 | 2297472 |         38946 | 2019-12-11 12:28:18.809582+00
        10 |         1 | 2297475 |         38459 | 2019-12-11 12:28:21.321489+00
(3 rows)
```

## Now, over to you.

Let's stop there and look at what we've built.

* An architecture on Kubernetes with a database and scalable compute
* A CRUD API with CREATE and RETRIEVE implemented in Node.js
* Security on a per device level using symmetrical keys
* A working client that can be used today on any Raspberry Pi

It's over to you now to continue learning and building out the CRUD solution with OpenFaaS.

### Keep learning

From here, you should be able to build the DELETE and UPDATE operations on your own, if you think that they are suited to this use-case. Perhaps when a device is de-provisioned, it should be able to delete its own rows?

The device_key enabled access to read the records for a single device, but should there be a set of administrative keys, such that users can query the device_status records for any devices?

What else would you like to record from your fleet Raspberry Pis? Perhaps you can add the external and IP address of each host to the table and update the Python client to collect that? There are several free websites available that can provide this data.

