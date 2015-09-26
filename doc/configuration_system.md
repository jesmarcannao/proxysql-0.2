Configuration system
====================

ProxySQL has a complex but easy to use configuration system suited to serve the following needs:
* allow easy automated updates to the configuration (this is because some ProxySQL users use it in larger setups with automated provisioning). There is a MySQL-compatible admin interface for this purpose
* allow as many configuration items as possible to be modified at runtime, without restarting the daemon
* allow easy rollbacks of wrong configurations

This is achieved using a multi-layer configuration system where settings are moved from one layer to another.
The 3 layers of the configuration system are described in the picture below:

```
+-------------------------+
|         RUNTIME         |
+-------------------------+
       /|\          |
        |           |
    [1] |       [2] |
        |          \|/
+-------------------------+
|         MEMORY          |
+-------------------------+ _
       /|\          |      |\
        |           |        \
    [3] |       [4] |         \ [5]
        |          \|/         \
+-------------------------+  +-------------------------+
|          DISK           |  |       CONFIG FILE       |
+-------------------------+  +-------------------------+

```

__RUNTIME__ represents the in-memory data structures of ProxySQL used by the threads that are handling the requests. These contains the values of the global variables used, the list of backend servers grouped into hostgroups or the list of MySQL users that can connect to the proxy. Note that operators can never modify the contents of the __RUNTIME__ configuration section directly. They always have to go through the bottom layers.

__MEMORY__ (sometime also referred as *main*) represents an in-memory SQLite3 database which is exposed to the outside via a MySQL-compatible interface. Users can connect with a MySQL client to this interface and query different tables and databases. The configuration tables available through this interface are:
* mysql_servers -- the list of backend servers
* mysql_users -- the list of users and their credentials which can connect to ProxySQL. Note that ProxySQL will use these credentials to connect to the backend servers as well
* mysql_query_rules -- the list of rules for routing traffic to the different backend servers. These rules can also cause a rewrite of the query, or caching of the result
* global_variables -- the list of global variables used throughout the proxy that can be tweaked at runtime. Examples of global variables:
```
mysql> select * from global_variables limit 3;
+----------------------------------+----------------+
| variable_name                    | variable_value |
+----------------------------------+----------------+
| mysql-connect_retries_on_failure | 5              |
| mysql-connect_retries_delay      | 1              |
| mysql-connect_timeout_server_max | 10000          |
+----------------------------------+----------------+
```
* mysql_collations -- the list of MySQL collations available for the proxy to work with. These are extracted directly from the client library.
* [only available in debug builds] debug_levels -- the list of types of debug statements that ProxySQL emits together with their verbosity levels. This allows us to easily configure at runtime what kind of statements we have in the log in order to debug different problems. This is available only in debug builds because it can affect performance 

__DISK__ and __CONFIG FILE__

__DISK__ represents an on-disk SQLite3 database, with the default location at `$(DATADIR)/proxysql.db`. Across restarts, the in-memory configs that were not persisted will be lost, therefore it is important to persist the configuration into __DISK__ . __CONFIG__ file is the classical config file, and we'll see the relationship between it and the other configuration layers in the next section.

In the following sections, we'll describe the lifecycle of each of these layers for the basic operations that the daemon goes through: starting up for the first time, starting up, restarting, shutting down, etc.

# Startup

At a normal start-up, ProxySQL reads its config file (if present) to determine its datadir.
What happen next depends from the presence or not of its database file (disk) in its datadir.

If the database file is found, ProxySQL initializes its in-memory configuration from the persisted on-disk database. So, disk gets loaded into memory and then propagated towards the runtime configuration.
If the database file is not found and a config file exists, the config file is parsed and its content is loaded into the in-memory database, to then be both saved on-disk database and loaded at runtime.
If it important to note that **if a database file is found, the config file is not parsed** is not parsed** . That is, during a normal start-up, ProxySQL initializes its in-memory configuration from the persisted on-disk database ONLY. 


# Initial startup (or --initial flag)

At the initial start-up, the memory and runtime configuration gets populated from the config file and not from database file.
It is possible to force an initial startup running proysql with --initial flag, which resets the database by renaming the old one.

After this is done, the configuration is also persisted to the disk database, which will be used for the next restarts.

# Reload startup (or --reload flag)

If proxysql is executed with the --reload flag, it attempts to merge the configuration in the config file with the content of the database file. After that, it performs a regular startup.
There is no guarantee that ProxySQL will successfully manage to merge the two configuration source if they conflicts, and user should validate that the merge was as expected.


# Modifying config at runtime

Modifying the config at runtime is done through the MySQL admin port of ProxySQL. After connecting to it, we are presented with a MySQL-compatible interface for querying several ProxySQL-related tables:
```mysql
mysql> show tables;
+-------------------+
| tables            |
+-------------------+
| mysql_servers     |
| mysql_users       |
| mysql_query_rules |
| global_variables  |
| mysql_collations  |
| debug_levels      |
+-------------------+
6 rows in set (0.01 sec)
```

Each such table has a well defined role in the admin interface:
- `mysql_servers` contains the list of backend servers for ProxySQL to connect to
- `mysql_users` contains the list of users with which to authenticate to ProxySQL and the backend servers
- `mysql_query_rules` contains the rules for caching, routing or rewriting the SQL queries that pass through the proxy
- `global_variables` contains both the MySQL variables and the admin variables in a single table
- `debug_levels` is only used in the debug build of ProxySQL

These tables represent the middle layer (in-memory database) from the diagram above and can be manipulated using standard SQL queries. In order to move the configuration from this layer upwards or downwards, see the next section.

For more details about these tables and their fields, see their dedicated description in the documentation.

# Moving config between layers

In order to move configuration between the three layers, there are a set of different admin commands available through the admin interface. Once you understand what each of the three layers means, the semantics should be quite obvious. Together with the explanation of each command, there is a number written next to it. The number corresponds to the arrow in the diagram from above.

For handling MySQL users:
* [1] LOAD MYSQL USERS FROM MEMORY / LOAD MYSQL USERS TO RUNTIME
  * loads MySQL users from the in-memory database to the runtime data structures
* [2] SAVE MYSQL USERS TO MEMORY / SAVE MYSQL USERS FROM RUNTIME
  * persists the MySQL users from the runtime data structures to the in-memory database
* [3] LOAD MYSQL USERS TO MEMORY / LOAD MYSQL USERS FROM DISK
  * loads MySQL users from the on-disk database to the in-memory database
* [4] SAVE MYSQL USERS FROM MEMORY / SAVE MYSQL USERS TO DISK
  * persists the MySQL users from the in-memory database to the on-disk database
* [5] LOAD MYSQL USERS FROM CONFIG
  * loads from the configuration file the users into the in-memory database

For handling MySQL servers:
* [1] LOAD MYSQL SERVERS FROM MEMORY / LOAD MYSQL SERVERS TO RUNTIME
  * loads MySQL servers from the in-memory database to the runtime data structures
* [2] SAVE MYSQL SERVERS TO MEMORY / SAVE MYSQL SERVERS FROM RUNTIME
  * persists the MySQL servers from the runtime data structures to the in-memory database
* [3] LOAD MYSQL SERVERS TO MEMORY / LOAD MYSQL SERVERS FROM DISK
  * loads MySQL servers from the on-disk database to the in-memory database
* [4] SAVE MYSQL SERVERS FROM MEMORY / SAVE MYSQL SERVERS TO DISK
  * persists the MySQL servers from the in-memory database to the on-disk database
* [5] LOAD MYSQL SERVERS FROM CONFIG
  * loads from the configuration file the servers into the in-memory database

For handling MySQL query rules:
* [1] LOAD MYSQL QUERY RULES FROM MEMORY / LOAD MYSQL QUERY RULES TO RUNTIME
  * loads MySQL query rules from the in-memory database to the runtime data structures
* [2] SAVE MYSQL QUERY RULES TO MEMORY / SAVE MYSQL QUERY RULES FROM RUNTIME
  * persists the MySQL query rules from the runtime data structures to the in-memory database
* [3] LOAD MYSQL QUERY RULES TO MEMORY / LOAD MYSQL QUERY RULES FROM DISK
  * loads MySQL query rules from the on-disk database to the in-memory database
* [4] SAVE MYSQL QUERY RULES FROM MEMORY / SAVE MYSQL QUERY RULES TO DISK
  * persists the MySQL query rules from the in-memory database to the on-disk database
* [5] LOAD MYSQL QUERY RULES FROM CONFIG
  * loads from the configuration file the query rules into the in-memory database

For handling MySQL variables:
* [1] LOAD MYSQL VARIABLES TO MEMORY / LOAD MYSQL VARIABLES FROM DISK
  * loads MySQL variables from the on-disk database to the in-memory database
* [2] SAVE MYSQL VARIABLES FROM MEMORY / SAVE MYSQL VARIABLES TO DISK
  * persists the MySQL variables from the in-memory database to the on-disk database
* [3] LOAD MYSQL VARIABLES FROM MEMORY / LOAD MYSQL VARIABLES TO RUNTIME
  * loads MySQL variables from the in-memory database to the runtime data structures
* [4] SAVE MYSQL VARIABLES TO MEMORY / SAVE MYSQL VARIABLES FROM RUNTIME
  * persists the MySQL variables from the runtime data structures to the in-memory database
* [5] LOAD MYSQL VARIABLES FROM CONFIG
  * loads from the configuration file the variables into the in-memory database

For handling admin variables:
* [1] LOAD ADMIN VARIABLES FROM MEMORY / LOAD ADMIN VARIABLES TO RUNTIME
  * loads admin variables from the in-memory database to the runtime data structures
* [2] SAVE ADMIN VARIABLES TO MEMORY / SAVE ADMIN VARIABLES FROM RUNTIME
  * persists the admin variables from the runtime data structures to the in-memory database
* [3] LOAD ADMIN VARIABLES TO MEMORY / LOAD ADMIN VARIABLES FROM DISK
  * loads admin variables from the on-disk database to the in-memory database
* [4] SAVE ADMIN VARIABLES FROM MEMORY / SAVE ADMIN VARIABLES TO DISK
  * persists the admin variables from the in-memory database to the on-disk database

Note: the above command allows the following shortname :
* **MEM** for **MEMORY**
* **RUN** for **RUNTIME**

So, for example, these two commands are equivalent:
* SAVE ADMIN VARIABLES TO MEMORY
* SAVE ADMIN VARIABLES TO MEM
