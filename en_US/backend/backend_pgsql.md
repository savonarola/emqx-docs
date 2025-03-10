# PostgreSQL Backend

::: tip

After EMQX version 3.1, a powerful rule engine is introduced to replace plug-ins. It is recommended that you use it. See [Save data to PostgreSQL](../rule/backend_pgsql.md) to setup Save data to PostgreSQL in rule engine.

:::

Config file: emqx\_backend\_pgsql.conf

## Configure PostgreSQL Server

Connection pool of multiple PostgreSQL servers is supported:

```bash
## Pgsql Server
backend.pgsql.pool1.server = 127.0.0.1:5432

## Pgsql Pool Size
backend.pgsql.pool1.pool_size = 8

## Pgsql Username
backend.pgsql.pool1.username = root

## Pgsql Password
backend.pgsql.pool1.password = public

## Pgsql Database
backend.pgsql.pool1.database = mqtt

## Pgsql Ssl
backend.pgsql.pool1.ssl = false

## Max number of fetch offline messages. Without count limit if infinity
## backend.pgsql.max_returned_count = 500

## Time Range. Without time limit if infinity
## d - day
## h - hour
## m - minute
## s - second
## backend.pgsql.time_range = 2h
```

## Configure PostgreSQL Persistence Hooks

```bash
## Client Connected Record
backend.pgsql.hook.client.connected.1    = {"action": {"function": "on_client_connected"}, "pool": "pool1"}

## Subscribe Lookup Record
backend.pgsql.hook.client.connected.2    = {"action": {"function": "on_subscribe_lookup"}, "pool": "pool1"}

## Client DisConnected Record
backend.pgsql.hook.client.disconnected.1 = {"action": {"function": "on_client_disconnected"}, "pool": "pool1"}

## Lookup Unread Message QOS > 0
backend.pgsql.hook.session.subscribed.1  = {"topic": "#", "action": {"function": "on_message_fetch"}, "pool": "pool1"}

## Lookup Retain Message
backend.pgsql.hook.session.subscribed.2  = {"topic": "#", "action": {"function": "on_retain_lookup"}, "pool": "pool1"}

## Store Publish Message  QOS > 0
backend.pgsql.hook.message.publish.1     = {"topic": "#", "action": {"function": "on_message_publish"}, "pool": "pool1"}

## Store Retain Message
backend.pgsql.hook.message.publish.2     = {"topic": "#", "action": {"function": "on_message_retain"}, "pool": "pool1"}

## Delete Retain Message
backend.pgsql.hook.message.publish.3     = {"topic": "#", "action": {"function": "on_retain_delete"}, "pool": "pool1"}

## Store Ack
backend.pgsql.hook.message.acked.1       = {"topic": "#", "action": {"function": "on_message_acked"}, "pool": "pool1"}

## Get offline messages
### "offline_opts": Get configuration for offline messages
### max_returned_count: Maximum number of offline messages get at a time
### time_range: Get only messages in the current time range
## backend.pgsql.hook.session.subscribed.1  = {"topic": "#", "action": {"function": "on_message_fetch"}, "offline_opts": {"max_returned_count": 500, "time_range": "2h"}, "pool": "pool1"}

## If you need to store Qos0 messages, you can enable the following configuration
## Warning: When the following configuration is enabled, 'on_message_fetch' needs to be disabled, otherwise qos1, qos2 messages will be stored twice
## backend.pgsql.hook.message.publish.4     = {"topic": "#", "action": {"function": "on_message_store"}, "pool": "pool1"}
```

## Description of PostgreSQL Persistence Hooks

| hook                | topic | action                   | Description                     |
| ------------------- | ----- | ------------------------ | ------------------------------- |
| client.connected    |       | on\_client\_connected    | Store client connected state    |
| client.connected    |       | on\_subscribe\_lookup    | Subscribed topics               |
| client.disconnected |       | on\_client\_disconnected | Store client disconnected state |
| session.subscribed  | \#    | on\_message\_fetch       | Fetch offline messages          |
| session.subscribed  | \#    | on\_retain\_lookup       | Lookup retained messages        |
| message.publish     | \#    | on\_message\_publish     | Store published messages        |
| message.publish     | \#    | on\_message\_retain      | Store retained messages         |
| message.publish     | \#    | on\_retain\_delete       | Delete retained messages        |
| message.acked       | \#    | on\_message\_acked       | Process ACK                     |

## SQL Parameters Description

| hook                 | Parameters                           | Example (${name} represents available parameter)               |
| -------------------- | ------------------------------------ | -------------------------------------------------------------- |
| client.connected     | clientid                             | insert into conn(clientid) values(${clientid})                 |
| client.disconnected  | clientid                             | insert into disconn(clientid) values(${clientid})              |
| session.subscribed   | clientid, topic, qos                 | insert into sub(topic, qos) values(${topic}, ${qos})           |
| session.unsubscribed | clientid, topic                      | delete from sub where topic = ${topic}                         |
| message.publish      | msgid, topic, payload, qos, clientid | insert into msg(msgid, topic) values(${msgid}, ${topic})       |
| message.acked        | msgid, topic, clientid               | insert into ack(msgid, topic) values(${msgid}, ${topic})       |
| message.delivered    | msgid, topic, clientid               | insert into delivered(msgid, topic) values(${msgid}, ${topic}) |

## Configure 'action' with SQL

PostgreSQL backend supports SQL in
'action':

```bash
## After a client is connected to the EMQX server, it executes a SQL command (multiple command also supported)
backend.pgsql.hook.client.connected.3 = {"action": {"sql": ["insert into conn(clientid) values(${clientid})"]}, "pool": "pool1"}
```

## Create PostgreSQL DB

```bash
createdb mqtt -E UTF8 -e
```

## Import PostgreSQL DB & Table Schema

```bash
\i etc/sql/emqx_backend_pgsql.sql
```

::: tip
DB name is free of choice
:::
## PostgreSQL Client Connection Table

*mqtt\_client* stores client connection states:

```sql
CREATE TABLE mqtt_client(
    id SERIAL primary key,
    clientid character varying(100),
    state integer,
    node character varying(100),
    online_at timestamp,
    offline_at timestamp,
    created timestamp without time zone,
    UNIQUE (clientid)
);
```

Query a client's connection state:

```sql
select * from mqtt_client where clientid = ${clientid};
```

E.g., if client 'test' is online:

```bash
select * from mqtt_client where clientid = 'test';

id | clientid | state | node             | online_at           | offline_at        | created
----+----------+-------+----------------+---------------------+---------------------+---------------------
 1 | test     | 1     | emqx@127.0.0.1 | 2016-11-15 09:40:40 | NULL                | 2016-12-24 09:40:22
(1 rows)
```
Client 'test' is offline:

```bash
select * from mqtt_client where clientid = 'test';

    id | clientid | state | nod            | online_at           | offline_at          | created
----+----------+-------+----------------+---------------------+---------------------+---------------------
    1 | test     | 0     | emqx@127.0.0.1 | 2016-11-15 09:40:40 | 2016-11-15 09:46:10 | 2016-12-24 09:40:22
(1 rows)
```

## PostgreSQL Subscription Table

*mqtt\_sub* stores subscriptions of clients:

```sql
CREATE TABLE mqtt_sub(
    id SERIAL primary key,
    clientid character varying(100),
    topic character varying(200),
    qos integer,
    created timestamp without time zone,
    UNIQUE (clientid, topic)
);
```
E.g., client 'test' subscribes to topic 'test\_topic1' and
'test\_topic2':

```sql
insert into mqtt_sub(clientid, topic, qos) values('test', 'test_topic1', 1);
insert into mqtt_sub(clientid, topic, qos) values('test', 'test_topic2', 2);
```

Query subscription of a client:

```sql
select * from mqtt_sub where clientid = ${clientid};
```
Query subscription of client 'test':

```bash
select * from mqtt_sub where clientid = 'test';

    id | clientId     | topic       | qos  | created
----+--------------+-------------+------+---------------------
    1 | test         | test_topic1 |    1 | 2016-12-24 17:09:05
    2 | test         | test_topic2 |    2 | 2016-12-24 17:12:51
(2 rows)
```

## PostgreSQL Message Table

*mqtt\_msg* stores MQTT messages:

```sql
CREATE TABLE mqtt_msg (
  id SERIAL primary key,
  msgid character varying(60),
  sender character varying(100),
  topic character varying(200),
  qos integer,
  retain integer,
  payload text,
  arrived timestamp without time zone
);
```

Query messages published by a client:

```sql
select * from mqtt_msg where sender = ${clientid};
```
Query messages published by client 'test':

```bash
select * from mqtt_msg where sender = 'test';

    id | msgid                         | topic    | sender | node | qos | retain | payload | arrived
----+-------------------------------+----------+--------+------+-----+--------+---------+---------------------
    1  | 53F98F80F66017005000004A60003 | hello    | test   | NULL |   1 |      0 | hello   | 2016-12-24 17:25:12
    2  | 53F98F9FE42AD7005000004A60004 | world    | test   | NULL |   1 |      0 | world   | 2016-12-24 17:25:45
(2 rows)
```

## PostgreSQL Retained Message Table

*mqtt\_retain* stores retained messages:

```sql
CREATE TABLE mqtt_retain(
  id SERIAL primary key,
  topic character varying(200),
  msgid character varying(60),
  sender character varying(100),
  qos integer,
  payload text,
  arrived timestamp without time zone,
  UNIQUE (topic)
);
```

Query retained messages:

```sql
select * from mqtt_retain where topic = ${topic};
```
Query retained messages with topic 'retain':

```sql
select * from mqtt_retain where topic = 'retain';

    id | topic    | msgid                         | sender  | node | qos  | payload | arrived
----+----------+-------------------------------+---------+------+------+---------+---------------------
    1 | retain   | 53F33F7E4741E7007000004B70001 | test    | NULL |    1 | www     | 2016-12-24 16:55:18
(1 rows)
```

## PostgreSQL Acknowledgement Table

*mqtt\_acked* stores acknowledgements from the clients:

```sql
CREATE TABLE mqtt_acked (
  id SERIAL primary key,
  clientid character varying(100),
  topic character varying(100),
  mid integer,
  created timestamp without time zone,
  UNIQUE (clientid, topic)
);
```

## Enable PostgreSQL Backend

```bash
./bin/emqx_ctl plugins load emqx_backend_pgsql
```
