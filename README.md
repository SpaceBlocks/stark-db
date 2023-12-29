# stark-db

[![Build and test status](https://github.com/WeWatchWall/stark-db/workflows/Node.js%20CI/badge.svg)](https://github.com/WeWatchWall/stark-db/actions?query=workflow%3A%22Node.js+CI%22)
[![NPM version](https://img.shields.io/npm/v/stark-db.svg)](https://www.npmjs.com/package/stark-db)

SQLite-backed, change-tracking database available over HTTP.


## Installation

```bash
npm i -g stark-db
```

## Basics

Run with:

```bash
stark-db
```

By default, the DB engine is configured to run over SSL. While some self-signed
ssl certificates are automatically generated, they are not ever valid so
the user must supply their own. Then, the user needs to set the `-c` flag
in order to enable the cookie security. Only then is this system ready for
use in production.

There is a Swagger endpoint `https://127.0.0.1:5984/api-docs` where the user can
try out the routes available.

You may want to use `BEGIN IMMEDIATE TRANSACTION;` if you write to the database
concurrently as SQLite will throw busy errors otherwise.

This database tracks changes to all entities in the auto-created column(on all
tables) `stark_version`. There is also an extra table generated with any
user-created table called `_stark_del_${name}`. Deletions are tracked
in this auxiliary set of tables. With the help of this change tracking,
synchronization mechanisms can be built later. The user has the option
of using soft deletion -- marking data as deleted -- or relying on an
`id` column to track such deletions. ROWID would not work in a synchronization
scenario. Any modifications made by stark-db to the sqlite database
can be seen by running `select * from sqlite_master;` and the user can edit the
triggers/tables.
Also, the `-f` flag prevents all modifications related to change tracking.

If the user wants to see query results for every query, they need to run
one query at a time. Otherwise, only the results of the first query are shown.
Also, the change tracking API -- i.e. when creating or modifying tables -- 
expects one query at a time. Breaking down scripts into queries is not a simple
parsing job as parsing cannot rely on delimiters such as `;`.

Interactive queries are supported due to the stateful nature of the API.
A DB connection is marked inactive and is refreshed after 1 hour. A session
cookie to the server is marked inactive after 1 day.

## CLI

```bash
  -a, --address <address>  HTTP address to listen on (default: "127.0.0.1")
  -i, --doc <address>      Address to query by the documentation (default: "https://127.0.0.1")
  -p, --port <port>        HTTP port to listen on (default: "5983")
  -s, --ssl <port>         HTTPS port to listen on (default: "5984")
  -c, --cookie             Secure cookie, served over valid HTTPS only (default: false)
  -d, --data <path>        Path to the data directory (default: "./data")
  -k, --certs <path>       Path to the certs directory (default: "./certs")
  -f, --simple             Do not run change-tracking queries (default: false)
  -h, --help               display help for command
```

## HTTP API

### /{DB}/login

#### POST /{DB}/login

##### Summary: DB Login

##### Description: Logs into a database.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| DB | path | The database. | Yes | string |
| pid | query | The process ID. This is generated by the client. | Yes | string |

##### Body Example

```json
{
  "username": "admin",
  "password": "admin"
}
```

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The login was sucessful. |
| 401 | The login failed for the credentials. |
| 403 | The login failed for the DB. |

### /logout

#### POST /logout

##### Summary: User logout

##### Description:

Logs out of a user.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The logout was successful. |

### /users

#### GET /users

##### Summary: Get users

##### Description:

Get the users in the system. Regular users can only get their own user. Admins can get all the users.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |
| ID | query | The user ID. | No | string |
| name | query | The username. | No | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The requested user(s). |
| 401 | The user is not logged in. |
| 403 | The logged in user either doesn't have permission to view the user or the user wasn't found. |

#### POST /users

##### Summary: Add user

##### Description:

Add a user. Only Admins can do this.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |

##### Body Example

```json
{
  "username": "admin",
  "password": "admin",
  "salt": ""
}
```

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The user was added. |
| 401 | The user is not logged in. |
| 403 | The logged in user doesn't have permission to add the user or the user entity had an error. |

#### PUT /users

##### Summary:

Set user

##### Description:

Set a user. Regular users can only set their own user. Admins can set any user.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |

##### Body Example

```json
{
  "ID": 2,
  "username": "user1",
  "password": "password1",
  "salt": ""
}
```

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The user was set. |
| 401 | The user is not logged in. |
| 403 | The logged in user either doesn't have permission to set the user or the user entity was not found. |

#### DELETE /users

##### Summary: Delete user

##### Description:

Delete a user. Regular users can only delete their own user. Admins can delete any user.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |
| ID | query | The user ID | No | string |
| name | query | The username | No | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The user was deleted |
| 401 | The user is not logged in. |
| 403 | The logged in user either doesn't have permission to delete the user or the user wasn't found. |

### /DBs

#### GET /DBs

##### Summary: Get DBs

##### Description:

Get the DBs in the system. Regular users can only get their own DBs. Admins can get all the DBs.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |
| ID | query | The DB ID. | No | string |
| name | query | The DB name. | No | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The requested DB(s). |
| 401 | The user is not logged in. |
| 403 | The logged in user either doesn't have permission to get the DB or the DB wasn't found. |

#### POST /DBs

##### Summary: Add DB

##### Description:

Add a DB. Only Admins can do this.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |

##### Body Example

```json
{
  "name": "database1",
  "admins": [
    1,
    2
  ],
  "users": []
}
```

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The DB was added. |

#### PUT /DBs

##### Summary: Set DB

##### Description:

Set a DB. DB Admins can update their own DBs while Admins can update all DBs.

##### Body Example

```json
{
  "ID": 2,
  "name": "database1",
  "admins": [
    1,
    2
  ],
  "users": []
}
```

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The DB was set. |

#### DELETE /DBs

##### Summary: Delete DB

##### Description:

Delete a DB. DB Admins can delete their own DBs while Admins can delete any DB.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| pid | query | The process ID. This is generated by the client. | Yes | string |
| ID | query | The DB ID | No | string |
| name | query | The DB name | No | string |

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The DB was deleted |

### /{DB}/query

#### POST /{DB}/query

##### Summary: DB Query

##### Description:

Queries a database.

##### Parameters

| Name | Located in | Description | Required | Schema |
| ---- | ---------- | ----------- | -------- | ---- |
| DB | path | The database. | Yes | string |
| pid | query | The process ID. This is generated by the client. | Yes | string |

##### Body Example

```json
{
  "query": "SELECT * FROM sqlite_master;",
  "params": []
}
```

##### Responses

| Code | Description |
| ---- | ----------- |
| 200 | The successful query results. |
| 401 | The login failed for the credentials. |
| 403 | The login failed for the DB. |
| 500 | The server was not able to respond to the query. |