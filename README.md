          
      ,-#&%&%-,  ,-#&%&%-,  ,-#&%&#%-,  
     / .-----~  / .-----~  /___  ___/
     \ \____    \ \____       / /
      \_____`\   \_____`\    / /
     ______/ /  ______/ /   / /
    /_______/  /_______/   /_/

# SST (SQL Schema Transformer)

SST is a Common Lisp program that transforms an s-expression based database
schema format medium into appropriate SQL commands for specified RDBMS. SST is
capable of

* Dividing produced queries into logically separate (schemas, tables, primary keys, foreign keys, indexes) files.
* Customized (and extendable) identifier formatters. (You can adopt a naming scheme you prefer to name tables, indexes, etc.)
* `FOREIGN KEY` data type mismatch detection.
* Invalid `FOREIGN KEY` reference detection.
* Missing index detection for `ON DELETE`/`UPDATE CASCADE` references.
* `FOREIGN KEY` reference auto-completion. (You don't need to supply column of the referenced table. SST infers it for you.)

# Why SST?

SST is used to keep database schema consistency between various RDBMSes. At time
of this writing, SST is used to produce schema creation queries from an SST
medium containing ~200 tables in a CVS repository for PostgreSQL and Microsoft
SQL Server systems that are in production use. Every modification commited into
the SST medium in CVS repository gets propogated to production systems for
various RDBMes. Using such a scheme supplied below advantages:

* Unified synchronization between saveral R&D developments being worked on concurrently.
* Database schema modification history. (One can easily switch to a database schema branch of a point in time he/she wants.)
* Avoids writing RDBMS specific SQL queries. (One medium to rule them all!)

While there are [some other projects](http://sqlfairy.sourceforge.net/) on the
web that are (so called) capable of doing what SST is designed for, several
reasons (huge package dependency, complicated interface, lack of a proper
documentation, no (community) support, etc.) drived me to roll my own
solution.

# Installation

SST is `ASDF-INSTALL`able. On most modern Common Lisp implementations, ASDF
comes builtin. Assuming your Common Lisp implementation supports ASDF,
installation is just a single step:

    CL-USER> (asdf-install:install :sst)

# Example Usage

    $ cat /home/vy/projects/sst/examples/schema.lisp
    (with-schema ()
        "public"
      (with-table ()
          "companies"
          ((id serial :primary-key t)
           (name text :not-null t)))
      (with-table ()
          "units"
          (("id" (serial :start 10 :increment 2) :primary-key t)
           ("type" char :foreign-key ("types" :schema "meta"))
           ("code" float :default -3.4)
           ("company" text :foreign-key "companies"))
        (with-index (:unique t) "type")
        (with-index () ("code" :descending t) "type")))

    (with-schema ()
        "meta"
      (with-table ()
          "types"
          (("id" serial :primary-key t)
           ("name" text))))

    $ sbcl
    ...
    CL-USER> (asdf:oos 'asdf:load-op :cl-sst)
    ...
    CL-USER> (sst:produce-sql-output
    	  "/home/vy/projects/sst/examples/schema.lisp"
    	  "/home/vy/projects/sst/examples/pgsql/"
    	  'sst:rdbms-pgsql)

    WARNING: Expected (CHAR) data type doesn't match with the referenced (SERIAL)
    data type for constraint #<SQL-FOREIGN-KEY-CONSTRAINT {10038A6FA1}> on "type"
    column of "units" table in "public" schema.
    NIL
    CL-USER> (sst:produce-sql-output
    	  "/home/vy/projects/sst/examples/schema.lisp"
    	  "/home/vy/projects/sst/examples/mssql/"
    	  'sst:rdbms-mssql
    	  :identifier-case :upcase)
    WARNING: Expected (CHAR) data type doesn't match with the referenced (SERIAL)
    data type for constraint #<SQL-FOREIGN-KEY-CONSTRAINT {10047DA3C1}> on "type"
    column of "units" table in "public" schema.
    NIL

    $ cd /home/vy/projects/sst/examples

    $ ls -l pgsql/
    total 24
    -rw-r--r-- 1 vy vy 170 2008-09-20 14:31 foreign-keys
    -rw-r--r-- 1 vy vy 214 2008-09-20 16:49 foreign-keys.sql
    -rw-r--r-- 1 vy vy 122 2008-09-20 16:49 indexes.sql
    -rw-r--r-- 1 vy vy 216 2008-09-20 16:49 primary-keys.sql
    -rw-r--r-- 1 vy vy  42 2008-09-20 16:49 schemas.sql
    -rw-r--r-- 1 vy vy 294 2008-09-20 16:49 tables.sql

    $ cat pgsql/{schemas,tables,primary-keys,indexes,foreign-keys}.sql
    CREATE SCHEMA meta;
    CREATE SCHEMA public;
    CREATE TABLE meta.types (
        id serial,
        name text
    );
    CREATE TABLE public.companies (
        id serial,
        name text NOT NULL
    );
    CREATE TABLE public.units (
        id serial,
        type char,
        code float DEFAULT -3.4,
        company int
    );
    ALTER SEQUENCE public.units_id_seq START 10 INCREMENT 2;
    ALTER TABLE meta.types ADD CONSTRAINT types_id_pk PRIMARY KEY (id);
    ALTER TABLE public.companies ADD CONSTRAINT companies_id_pk PRIMARY KEY (id);
    ALTER TABLE public.units ADD CONSTRAINT units_id_pk PRIMARY KEY (id);
    CREATE INDEX units_code_desc_type_idx ON public.units (code DESC, type);
    CREATE UNIQUE INDEX units_type_unq ON public.units (type);
    ALTER TABLE public.units ADD CONSTRAINT units_company_fk FOREIGN KEY (company)
    REFERENCES public.companies (id);
    ALTER TABLE public.units ADD CONSTRAINT units_type_fk FOREIGN KEY (type)
    REFERENCES meta.types (id);

    $ ls -l mssql/
    total 20
    -rw-r--r-- 1 vy vy 218 2008-09-20 16:51 foreign-keys.sql
    -rw-r--r-- 1 vy vy 124 2008-09-20 16:51 indexes.sql
    -rw-r--r-- 1 vy vy 222 2008-09-20 16:51 primary-keys.sql
    -rw-r--r-- 1 vy vy  46 2008-09-20 16:51 schemas.sql
    -rw-r--r-- 1 vy vy 293 2008-09-20 16:51 tables.sql

    $ cat mssql/{schemas,tables,primary-keys,indexes,foreign-keys}.sql
    CREATE SCHEMA META
    GO
    CREATE SCHEMA PUBLIC
    GO
    CREATE TABLE META.TYPES (
        ID int IDENTITY(1,1),
        NAME varchar(max)
    )
    GO
    CREATE TABLE PUBLIC.COMPANIES (
        ID int IDENTITY(1,1),
        NAME varchar(max) NOT NULL
    )
    GO
    CREATE TABLE PUBLIC.UNITS (
        ID int IDENTITY(10,2),
        TYPE char,
        CODE float DEFAULT -3.4,
        COMPANY int
    )
    GO
    ALTER TABLE META.TYPES ADD CONSTRAINT PK_TYPES_ID PRIMARY KEY (ID)
    GO
    ALTER TABLE PUBLIC.COMPANIES ADD CONSTRAINT PK_COMPANIES_ID PRIMARY KEY (ID)
    GO
    ALTER TABLE PUBLIC.UNITS ADD CONSTRAINT PK_UNITS_ID PRIMARY KEY (ID)
    GO
    CREATE INDEX IX_UNITS_CODE_DESC_TYPE ON PUBLIC.UNITS (CODE DESC, TYPE)
    GO
    CREATE UNIQUE INDEX UK_UNITS_TYPE ON PUBLIC.UNITS (TYPE)
    GO
    ALTER TABLE PUBLIC.UNITS ADD CONSTRAINT FK_UNITS_COMPANY FOREIGN KEY (COMPANY)
    REFERENCES PUBLIC.COMPANIES (ID)
    GO
    ALTER TABLE PUBLIC.UNITS ADD CONSTRAINT FK_UNITS_TYPE FOREIGN KEY (TYPE)
    REFERENCES META.TYPES (ID)
    GO

# Syntax

    (PRODUCE-SQL-OUTPUT
      PATHNAME OUTPUT-DIRECTORY RDBMS
      &KEY IDENTIFIER-CASE SCHEMA-OUTPUT-FILE TABLE-OUTPUT-FILE
           PRIMARY-KEY-OUTPUT-FILE FOREIGN-KEY-OUTPUT-FILE INDEX-OUTPUT-FILE)

> `PRODUCE-SQL-OUTPUT` expands macros in the SST medium pointed by `PATHNAME`. It places output files under `OUTPUT-DIRECTORY`. Produced commands will be specific to given RDBMS and identifiers are case converted according to `IDENTIFIER-CASE`. (Valid values for `IDENTIFIER-CASE` are `:DOWNCASE`, `:UPCASE`, and `:QUOTE`.)

    (WITH-SCHEMA ATTRIBUTES SCHEMA-NAME &REST WITH-TABLE-COMPONENTS)
    (WITH-TABLE ATTRIBUTES TABLE-NAME COLUMNS &REST WITH-INDEX-COMPONENTS)

> List of `COLUMNS` forms are gets transformed to `WITH-COLUMN` expressions.

    (WITH-COLUMN COLUMN-NAME DATA-TYPE
      &KEY NOT-NULL DEFAULT PRIMARY-KEY FOREIGN-KEY)

> `DATA-TYPE`, `PRIMARY-KEY`, and `FOREIGN-KEY` forms are respectively transformed into `WITH-DATA-TYPE`, `WITH-PRIMARY-KEY`, and `WITH-FOREIGN-KEY` expressions.


    (WITH-DATA-TYPE DATA-TYPE &REST ATTRIBUTES)

> Supported data-types and relevant attributes are

    (BIGINT)
    (BIGSERIAL &KEY START INCREMENT)
    (BIT &OPTIONAL SIZE)
    (CHAR &OPTIONAL SIZE &KEY UNICODE)
    (FLOAT)
    (INT)
    (SERIAL &KEY START INCREMENT)
    (SMALLINT)
    (TEXT &KEY UNICODE)
    (TIMESTAMP &KEY WITH-TIME-ZONE)
    (VARCHAR &OPTIONAL SIZE &KEY UNICODE)

> Caveats are
>
> - At the moment, `UNICODE` key is meaningful only to Microsoft SQL Server.
> - At the moment, `WITH-TIME-ZONE` key is meaningful only to PostgreSQL.
>
> If one won't use any attributes of a data type, using list representation is optional.

    (WITH-FOREIGN-KEY TABLE &KEY SCHEMA ON-DELETE-CASCADE ON-UPDATE-CASCADE)

> Unless `SCHEMA` of the referenced `TABLE` is not in the current schema of current parsing scope, it's optional to supply a `SCHEMA`.

    (WITH-INDEX (&KEY UNIQUE) COLUMNS)
