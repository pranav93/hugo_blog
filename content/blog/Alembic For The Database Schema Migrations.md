---
title: "Alembic For The Database Schema Migrations"
date: 2019-11-09T19:01:09+05:30
draft: true
tags: ["mysql", "postgresql", "database", "python", "migration"]
---

Using just sql scripts for managing database schema migrations is alright.
But in that case we have to explicitly create a system where we have to 
keep track of the sequences of the scripts (one of the reasons: foreign key dependencies).
While in development environment, if we want to check whether existing 
function/method code compatible with earlier table schema, we have to
manually write some commands in our favorite db client. That adds up extra effort.

There are many ways we can automate the process of upgrading/rollbacking the db changes by 
using tools like flyway/liquibase.
But I find it quite tedious to use json/yaml and likewise for managing db migrations. Alembic does the
same job with Python code (which is simple to read and maintain).  
 
Let's go through an example to understand how awesomely easy it is to manage database migrations in alembic.
Let's create a directory `my_db_migration`, and `cd` into it
```bash
pranav@ubuntu:~$ mkdir my_db_migration
pranav@ubuntu:~$ cd my_db_migration
pranav@ubuntu:~/my_db_migration$
```

We can create a virtual environment for our project called `env`.
```bash
pranav@ubuntu:~/my_db_migration$ virtualenv env
Using base prefix '/usr'
New python executable in /home/pranav/my_db_migration/env/bin/python3
Also creating executable in /home/pranav/my_db_migration/env/bin/python
Installing setuptools, pip, wheel...
done.
```

Now, let's activate it,
```bash
pranav@ubuntu:~/my_db_migration$ source env/bin/activate
(env) pranav@ubuntu:~/my_db_migration$
```

We are ready to create our alembic project. For that we will install the `alembic` library with pip.
```bash
(env) pranav@ubuntu:~/my_db_migration$ pip install alembic
Collecting alembic
  Downloading https://files.pythonhosted.org/packages/70/3d/d5ed7a71fe84f9ed0a69e91232a40b0b148b151524dc5bb1c8e4211eb117/alembic-1.3.0.tar.gz (1.1MB)
     |████████████████████████████████| 1.1MB 923kB/s 
Collecting SQLAlchemy>=1.1.0
  Downloading https://files.pythonhosted.org/packages/14/0e/487f7fc1e432cec50d2678f94e4133f2b9e9356e35bacc30d73e8cb831fc/SQLAlchemy-1.3.10.tar.gz (6.0MB)
     |████████████████████████████████| 6.0MB 9.0MB/s 
Collecting Mako
  Downloading https://files.pythonhosted.org/packages/b0/3c/8dcd6883d009f7cae0f3157fb53e9afb05a0d3d33b3db1268ec2e6f4a56b/Mako-1.1.0.tar.gz (463kB)
     |████████████████████████████████| 471kB 1.8MB/s 
Collecting python-editor>=0.3
  Downloading https://files.pythonhosted.org/packages/c6/d3/201fc3abe391bbae6606e6f1d598c15d367033332bd54352b12f35513717/python_editor-1.0.4-py3-none-any.whl
Collecting python-dateutil
  Downloading https://files.pythonhosted.org/packages/d4/70/d60450c3dd48ef87586924207ae8907090de0b306af2bce5d134d78615cb/python_dateutil-2.8.1-py2.py3-none-any.whl (227kB)
     |████████████████████████████████| 235kB 8.0MB/s 
Collecting MarkupSafe>=0.9.2
  Downloading https://files.pythonhosted.org/packages/b2/5f/23e0023be6bb885d00ffbefad2942bc51a620328ee910f64abe5a8d18dd1/MarkupSafe-1.1.1-cp36-cp36m-manylinux1_x86_64.whl
Collecting six>=1.5
  Downloading https://files.pythonhosted.org/packages/65/26/32b8464df2a97e6dd1b656ed26b2c194606c16fe163c695a992b36c11cdf/six-1.13.0-py2.py3-none-any.whl
Building wheels for collected packages: alembic, SQLAlchemy, Mako
  Building wheel for alembic (setup.py) ... done
  Created wheel for alembic: filename=alembic-1.3.0-py2.py3-none-any.whl size=144427 sha256=ca5d1444658877ebb2d612420ae10b6807ff190199ec1746ee67d80e997a8999
  Stored in directory: /home/pranav/.cache/pip/wheels/40/f8/22/ad0f408796a4c656fae5ee1fd8d8a139b19ca4af61059cea5b
  Building wheel for SQLAlchemy (setup.py) ... done
  Created wheel for SQLAlchemy: filename=SQLAlchemy-1.3.10-cp36-cp36m-linux_x86_64.whl size=1200400 sha256=b2b67ee0f089bfb5972b0cc2a19b439e89badb4da8094d405a2808c51286f287
  Stored in directory: /home/pranav/.cache/pip/wheels/4b/b2/89/cd2231ee623987c605f049df55f40a3e4252ef6a15b94836c2
  Building wheel for Mako (setup.py) ... done
  Created wheel for Mako: filename=Mako-1.1.0-cp36-none-any.whl size=75363 sha256=3f73fac5d5e18ce52c37f5a87f8e2c46c9d71dde81c4938bf21603ec44be73ef
  Stored in directory: /home/pranav/.cache/pip/wheels/98/32/7b/a291926643fc1d1e02593e0d9e247c5a866a366b8343b7aa27
Successfully built alembic SQLAlchemy Mako
Installing collected packages: SQLAlchemy, MarkupSafe, Mako, python-editor, six, python-dateutil, alembic
Successfully installed Mako-1.1.0 MarkupSafe-1.1.1 SQLAlchemy-1.3.10 alembic-1.3.0 python-dateutil-2.8.1 python-editor-1.0.4 six-1.13.0
```

We will create an alembic project with following command
```bash
(env) pranav@ubuntu:~/my_db_migration$ alembic init alembic
  Creating directory /home/pranav/my_db_migration/alembic ...  done
  Creating directory /home/pranav/my_db_migration/alembic/versions ...  done
  Generating /home/pranav/my_db_migration/alembic.ini ...  done
  Generating /home/pranav/my_db_migration/alembic/env.py ...  done
  Generating /home/pranav/my_db_migration/alembic/script.py.mako ...  done
  Generating /home/pranav/my_db_migration/alembic/README ...  done
  Please edit configuration/connection/logging settings in '/home/pranav/my_db_migration/alembic.ini' before proceeding.
```

The above command creates a structure for alembic migrations. We have to edit the `.ini` file
to make the connection with the database. For this example we are going to use
the postgres database.

Let's connect to local postgres server on our dev machine using command

```bash
pranav@ubuntu:~$ psql
psql (10.10 (Ubuntu 10.10-0ubuntu0.18.04.1))
Type "help" for help.

pranav=>
```

Create a table called `example_db` for the example

```bash
pranav=> create database example_db;
CREATE DATABASE
pranav=> \c example_db;
You are now connected to database "example_db" as user "pranav".
```

Now that we have created the database `example_db`, let us edit the `alembic.ini` file
created by alembic to add the database connection string. replace 

`sqlalchemy.url = driver://user:pass@localhost/dbname`

with 

`sqlalchemy.url = postgres://username:password@localhost:5432/example_db`

Now, let's create a sqlalchemy model from which we are going to generate the database version.
For that we first have to install sqlalchemy.

```bash
(env) pranav@ubuntu:~/my_db_migration$ pip install sqlalchemy
Requirement already satisfied: sqlalchemy in ./env/lib/python3.6/site-packages (1.3.10)
(env) pranav@ubuntu:~/my_db_migration$ pip freeze
alembic==1.3.0
Mako==1.1.0
MarkupSafe==1.1.1
python-dateutil==2.8.1
python-editor==1.0.4
six==1.13.0
SQLAlchemy==1.3.10
```

Seems like it is already installed as it is the dependency for the `alembic`.

Create a models.py which contains the code,

```python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String

Base = declarative_base()


class Student(Base):
    __tablename__ = 'students'
    name = Column(String)
    seq = Column(Integer, primary_key=True)

    def __init__(self, name):
        self.name = name
```

Now that we have added our model, let's run some commands to autogenerate the migration.
```bash
(env) pranav@ubuntu:~/my_db_migration$ alembic revision --autogenerate -m "Added students table"
Traceback (most recent call last):
  File "/home/pranav/my_db_migration/env/bin/alembic", line 8, in <module>
    sys.exit(main())
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/config.py", line 575, in main
    CommandLine(prog=prog).main(argv=argv)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/config.py", line 569, in main
    self.run_cmd(cfg, options)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/config.py", line 549, in run_cmd
    **dict((k, getattr(options, k, None)) for k in kwarg)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/command.py", line 214, in revision
    script_directory.run_env()
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/script/base.py", line 489, in run_env
    util.load_python_file(self.dir, "env.py")
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/util/pyfiles.py", line 98, in load_python_file
    module = load_module_py(module_id, path)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/util/compat.py", line 173, in load_module_py
    spec.loader.exec_module(module)
  File "<frozen importlib._bootstrap_external>", line 678, in exec_module
  File "<frozen importlib._bootstrap>", line 219, in _call_with_frames_removed
  File "alembic/env.py", line 77, in <module>
    run_migrations_online()
  File "alembic/env.py", line 62, in run_migrations_online
    poolclass=pool.NullPool,
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/sqlalchemy/engine/__init__.py", line 522, in engine_from_config
    return create_engine(url, **options)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/sqlalchemy/engine/__init__.py", line 479, in create_engine
    return strategy.create(*args, **kwargs)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/sqlalchemy/engine/strategies.py", line 87, in create
    dbapi = dialect_cls.dbapi(**dbapi_args)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/sqlalchemy/dialects/postgresql/psycopg2.py", line 737, in dbapi
    import psycopg2
ModuleNotFoundError: No module named 'psycopg2'
```

As we are trying to connect to postgresql, we need `psycopg2` installed. Let's install it.
```bash
(env) pranav@ubuntu:~/my_db_migration$ sudo apt-get install libpq-dev
[sudo] password for pranav: 
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Suggested packages:
  postgresql-doc-10
The following NEW packages will be installed:
  libpq-dev
0 upgraded, 1 newly installed, 0 to remove and 29 not upgraded.
Need to get 218 kB of archives.
After this operation, 1,092 kB of additional disk space will be used.
Get:1 http://us.archive.ubuntu.com/ubuntu bionic-updates/main amd64 libpq-dev amd64 10.10-0ubuntu0.18.04.1 [218 kB]
Fetched 218 kB in 1s (232 kB/s)    
Selecting previously unselected package libpq-dev.
(Reading database ... 192301 files and directories currently installed.)
Preparing to unpack .../libpq-dev_10.10-0ubuntu0.18.04.1_amd64.deb ...
Unpacking libpq-dev (10.10-0ubuntu0.18.04.1) ...
Setting up libpq-dev (10.10-0ubuntu0.18.04.1) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
(env) pranav@ubuntu:~/my_db_migration$ pip install psycopg2
Collecting psycopg2
  Using cached https://files.pythonhosted.org/packages/84/d7/6a93c99b5ba4d4d22daa3928b983cec66df4536ca50b22ce5dcac65e4e71/psycopg2-2.8.4.tar.gz
Building wheels for collected packages: psycopg2
  Building wheel for psycopg2 (setup.py) ... done
  Created wheel for psycopg2: filename=psycopg2-2.8.4-cp36-cp36m-linux_x86_64.whl size=419315 sha256=1de8e1dd0029846c82a21db758c76b774c71133aa3d124c1b40d950b9c6d9617
  Stored in directory: /home/pranav/.cache/pip/wheels/7e/5b/53/30085c62689dcfce50c8f40759945a49eb856af082e9ebf751
Successfully built psycopg2
Installing collected packages: psycopg2
Successfully installed psycopg2-2.8.4
```

Now, we are gonna create table using autogenerate command

```bash
(env) pranav@ubuntu:~/my_db_migration$ alembic revision --autogenerate -m "Added students table"
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
ERROR [alembic.util.messaging] Can't proceed with --autogenerate option; environment script /home/pranav/my_db_migration/alembic/env.py does not provide a MetaData object or sequence of objects to the context.
  FAILED: Can't proceed with --autogenerate option; environment script /home/pranav/my_db_migration/alembic/env.py does not provide a MetaData object or sequence of objects to the context.
```

But it fails. The reason is that alembic is not having any information about table metadata to generate
the script. So, we will add the base metadata in `env.py`.

```python
from logging.config import fileConfig

from sqlalchemy import engine_from_config
from sqlalchemy import pool

from alembic import context

from models import Base

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = Base.metadata

...
```

Now, let's run the autogenerate command again.

```
(env) pranav@ubuntu:~/my_db_migration$ alembic revision --autogenerate -m "Added students table"
Traceback (most recent call last):
  File "/home/pranav/my_db_migration/env/bin/alembic", line 8, in <module>
    sys.exit(main())
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/config.py", line 575, in main
    CommandLine(prog=prog).main(argv=argv)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/config.py", line 569, in main
    self.run_cmd(cfg, options)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/config.py", line 549, in run_cmd
    **dict((k, getattr(options, k, None)) for k in kwarg)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/command.py", line 214, in revision
    script_directory.run_env()
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/script/base.py", line 489, in run_env
    util.load_python_file(self.dir, "env.py")
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/util/pyfiles.py", line 98, in load_python_file
    module = load_module_py(module_id, path)
  File "/home/pranav/my_db_migration/env/lib/python3.6/site-packages/alembic/util/compat.py", line 173, in load_module_py
    spec.loader.exec_module(module)
  File "<frozen importlib._bootstrap_external>", line 678, in exec_module
  File "<frozen importlib._bootstrap>", line 219, in _call_with_frames_removed
  File "alembic/env.py", line 8, in <module>
    from models import Base
ModuleNotFoundError: No module named 'models'
```

It seems that current directory is not in PYTHONPATH. Add current directory to PYTHONPATH with command,

```bash
(env) pranav@ubuntu:~/my_db_migration$ export PYTHONPATH=.
(env) pranav@ubuntu:~/my_db_migration$ alembic revision --autogenerate -m "Added students table"
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added table 'students'
  Generating /home/pranav/my_db_migration/alembic/versions/6f1220842ecc_added_students_table.py ...  done
```

It successfully creates the version.

```bash
pranav@ubuntu:~/my_db_migration$ ls
alembic  alembic.ini  env  models.py  __pycache__
pranav@ubuntu:~/my_db_migration$ cd alembic/
pranav@ubuntu:~/my_db_migration/alembic$ ls
env.py  __pycache__  README  script.py.mako  versions
pranav@ubuntu:~/my_db_migration/alembic$ cd versions/
pranav@ubuntu:~/my_db_migration/alembic/versions$ ls
6f1220842ecc_added_students_table.py  __pycache__
pranav@ubuntu:~/my_db_migration/alembic/versions$ cat 6f1220842ecc_added_students_table.py 
"""Added students table

Revision ID: 6f1220842ecc
Revises: 
Create Date: 2019-11-09 18:59:29.147570

"""
from alembic import op
import sqlalchemy as sa


# revision identifiers, used by Alembic.
revision = '6f1220842ecc'
down_revision = None
branch_labels = None
depends_on = None


def upgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    op.create_table('students',
    sa.Column('name', sa.String(), nullable=True),
    sa.Column('seq', sa.Integer(), nullable=False),
    sa.PrimaryKeyConstraint('seq')
    )
    # ### end Alembic commands ###


def downgrade():
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_table('students')
    # ### end Alembic commands ###
```

Now, let's apply the migration to our database,

```bash
(env) pranav@ubuntu:~/my_db_migration$ alembic upgrade head
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 6f1220842ecc, Added students table
```

We have successfully applied the migration. To see the table created, go to the
postgres client and execute,

```bash
example_db=> \dt
             List of relations
 Schema |      Name       | Type  | Owner  
--------+-----------------+-------+--------
 public | alembic_version | table | pranav
 public | students        | table | pranav
(2 rows)

example_db=> \d students
                                  Table "public.students"
 Column |       Type        | Collation | Nullable |                Default                
--------+-------------------+-----------+----------+---------------------------------------
 name   | character varying |           |          | 
 seq    | integer           |           | not null | nextval('students_seq_seq'::regclass)
Indexes:
    "students_pkey" PRIMARY KEY, btree (seq)
```

You can check the current alembic version of your database,

```bash
example_db=> select * from alembic_version;
 version_num  
--------------
 6f1220842ecc
(1 row)

```

Let's add another non nullable column email to the table, and make name non nullable as well.

```python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String

Base = declarative_base()


class Student(Base):
    __tablename__ = 'students'
    name = Column(String, nullable=False)
    seq = Column(Integer, primary_key=True)
    email = Column(String, nullable=False)

    def __init__(self, name, email):
        self.name = name
        self.email = email
```

Let's run the autogenerate command and apply the migration,

```bash
(env) pranav@ubuntu:~/my_db_migration$ alembic revision --autogenerate -m "Added column 'email' to students table, and made 'name' necessary"
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.ddl.postgresql] Detected sequence named 'students_seq_seq' as owned by integer column 'students(seq)', assuming SERIAL and omitting
INFO  [alembic.autogenerate.compare] Detected added column 'students.email'
INFO  [alembic.autogenerate.compare] Detected NOT NULL on column 'students.name'
  Generating /home/pranav/my_db_migration/alembic/versions/64d7e64488b4_added_column_email_to_students_table_.py ...  done
(env) pranav@ubuntu:~/my_db_migration$ alembic upgrade head
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 6f1220842ecc -> 64d7e64488b4, Added column 'email' to students table, and made 'name' necessary
```

Now, if you see the students table structure again, 'name' is the mandatory field, and there
is new mandatory field 'email'. The alembic version is increased as well

```bash
example_db=> \d students 
                                  Table "public.students"
 Column |       Type        | Collation | Nullable |                Default                
--------+-------------------+-----------+----------+---------------------------------------
 name   | character varying |           | not null | 
 seq    | integer           |           | not null | nextval('students_seq_seq'::regclass)
 email  | character varying |           | not null | 
Indexes:
    "students_pkey" PRIMARY KEY, btree (seq)

example_db=> select * from alembic_version;
 version_num  
--------------
 64d7e64488b4
(1 row)

```

Want to rollback by one step? Just execute,

```bash
(env) pranav@ubuntu:~/my_db_migration$ alembic downgrade -1
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running downgrade 64d7e64488b4 -> 6f1220842ecc, Added column 'email' to students table, and made 'name' necessary
```

And voila,

```bash
example_db=> \d students
                                  Table "public.students"
 Column |       Type        | Collation | Nullable |                Default                
--------+-------------------+-----------+----------+---------------------------------------
 name   | character varying |           |          | 
 seq    | integer           |           | not null | nextval('students_seq_seq'::regclass)
Indexes:
    "students_pkey" PRIMARY KEY, btree (seq)

example_db=> select * from alembic_version;
 version_num  
--------------
 6f1220842ecc
(1 row)

```

As simple as that.
