---
layout: post
title:  "Least privilege for postgres users"
date:   2019-09-10T20:00:00+12:00
categories: Blog postgres
socialShareImage: /assets/images/2019-09/2019-09-03-postgres-dotnet-aws.png
disqus: true
description: "As a general rule, you should avoid running processes as root, or with system admin rights. If your process is compromised, running as an unprivileged user provides defense in depth making it much harder for an attacker to move through your system.
Likewise for databases, if your application doesn't require write access to the database, then it should connect with a user that doesn't have privileges to mutate data."
---

This is the second part in a series on using dotnet core with Amazon RDS for Postres. There is a [github repo with a sample project](https://github.com/williamdenton/AwsRdsPostgresDemo) that I'll be pulling code samples from. 

This series will cover:
1. [EfCore read-only Database Context](posts/2019/AWS-RDS-Postgres-npgsql-and-efcore-Part-1-ReadOnly-DbContext)
2. Configuring a local dev Postgres instance to emulate RDS (this post)
3. Authenticating to RDS using IAM
4. RDS failure modes with DotNet
5. `DbContext` gotchas with `IHostedService` and dependency injection

# Part 2 - Least privilege for postgres users

As a general rule, you should avoid running processes as root, or with system admin rights. If your process is compromised, running as an unprivileged user provides defense in depth making it much harder for an attacker to move through your system.

Likewise for databases, if your application doesn't require write access to the database, then it should connect with a user that doesn't have privileges to mutate data.

I consider these three roles to be the minimum configuration:
1. Migrator
2. Read-write
3. Read-only

Having a read-only role allows you to be able to connect to your database cluster using a non-primary node and run queries, this leaves your primary db node to the import job of writing data. If you're using AWS RDS for postgres the read-only is probably unnecessary as connecting to a read-only endpoint has the same effect.

## Roles

### Migrator
You need a role that is capable of executing DDL (Data Definition Language) to create your schema. DDL covers sql statements like `create`, `alter` and `drop`. These commands are not run during the normal course of your applications execution. Usually they run just before your application starts. A separate process in your deployment pipeline is responsible for running migrations and creating the schema your app needs to run.


### Read-Write
This is your normal database user, it can run DML (Data Manipulation Language) statements like `select`, `update` or `delete`, but can't run DDL.


### Read-Only
The read-only user can only run `select` statements.


## Postgres Setup
I've added a sample script in the demo application that goes with this series of posts on github [/williamdenton/AwsRdsPostgresDemo](https://github.com/williamdenton/AwsRdsPostgresDemo/docker/postgres/postgres-init/provision_demo_db_and_users.sql).


### Step 1 - Prepare your DB.
Stop everyone being able to see your stuff
```sql
revoke all on schema public from public;
revoke all on all tables in schema public from public;
```

Create the Database
```sql
CREATE DATABASE demo WITH OWNER = postgres;
REVOKE CONNECT ON DATABASE demo FROM PUBLIC;
```

connect to your new db to run the remaining steps in the context of the new database.
```sql
\connect demo
```


### Step 2 - Create Migrator
This creates the migrator user as the owner of the schema in the DB. While the Postgres user is the super user, the migrator user we create here is the next most powerful role. 

```sql
CREATE ROLE demo_migrator WITH LOGIN PASSWORD 'demo_migrator123' NOSUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION VALID UNTIL 'infinity';
GRANT CONNECT ON DATABASE demo TO demo_migrator;
CREATE SCHEMA service AUTHORIZATION demo_migrator;
ALTER ROLE demo_migrator SET search_path TO service;

GRANT ALL PRIVILEGES ON SCHEMA service TO demo_migrator;
```


### Step 3 - Create Read-Write User
Now we can create the read-write user. Importantly we set the `DEFAULT PRIVILEGES` for this user to be able to access the schema objects that the `demo_migrator` will create in the future.

```sql
CREATE ROLE demo_readwrite WITH LOGIN PASSWORD 'demo_readwrite123' NOSUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION VALID UNTIL 'infinity';
GRANT CONNECT ON DATABASE demo TO demo_readwrite;
ALTER ROLE demo_readwrite SET search_path TO service;
GRANT USAGE on SCHEMA service to demo_readwrite;

ALTER DEFAULT PRIVILEGES for user demo_migrator IN SCHEMA service GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO demo_readwrite;
ALTER DEFAULT PRIVILEGES for user demo_migrator IN SCHEMA service GRANT USAGE, SELECT ON SEQUENCES TO demo_readwrite;
ALTER DEFAULT PRIVILEGES for user demo_migrator IN SCHEMA service GRANT EXECUTE ON FUNCTIONS TO demo_readwrite;
```


### Step 4 - Create the Read-Only User
The read-only user looks very similar to the read-write. It is missing the last two grants that the read-write user has. I also default the user into a read only transaction. This gives better error messages in your app if you do accidentally attempt to write:
>`cannot execute UPDATE in a read-only transaction`

vs

>`permission denied for relation customers`

or

>`relation "customers" does not exist`

```sql
CREATE ROLE demo_readonly WITH LOGIN PASSWORD 'demo_readonly123' NOSUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION VALID UNTIL 'infinity';
GRANT CONNECT ON DATABASE demo TO demo_readonly;
GRANT USAGE on SCHEMA service to demo_readonly;
ALTER ROLE demo_readonly SET search_path TO service;

ALTER DEFAULT PRIVILEGES for user demo_migrator IN SCHEMA service GRANT SELECT ON TABLES TO demo_readonly;
ALTER USER demo_readonly set default_transaction_read_only = on;
```


## Passwords
Obviously your password should be a cryptographically random string and not `username123` on a real database. This script is not intended to harden your production server, I am not a DBA, and I am sure to have overlooked some critical detail. But for local development, having separate users with different passwords to connect with proves our application is connecting correctly.

>The issue of maintaining passwords for postgres users will be covered in the next post in the series, using IAM authentication in AWS.

## Bootstrap the db and users in Docker
If you place this script in a folder named  `/docker-entrypoint-initdb.d`, postgres will automatically execute it on its first startup.
I've found the most reliable way to do this is to build a [docker image with this file baked in](https://github.com/williamdenton/AwsRdsPostgresDemo/tree/master/docker/postgres). Alternatively you can mount a volume but I've had trouble making this work 100% of the time on windows.


## Wrap up
Always run your applications with the minimum level of access required to function. Aside from the obvious security benefits, this makes your application easier to reason about when changes are required, and in the case of read-only users, allows your application to scale more easily to make use of your non-primary database nodes in your postgres cluster.

In this post I covered why you should use at least three users in your postgres database. How to script these users. And a way to get postgres to automatically execute the script on startup.

The code samples in this post are available in full on github [https://github.com/williamdenton/AwsRdsPostgresDemo](https://github.com/williamdenton/AwsRdsPostgresDemo)
