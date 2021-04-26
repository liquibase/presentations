# Percona Live 2021
# Databases: The Anchor in your CI/CD Process

Recorded Presentation: TBD

DevOps is about improving processes to develop and deliver quality software with both speed and stability. At face value, it's a simple concept. However, fear of instability and the desire to control databases are preventing many organizations from updating their process. 

The fear of changing processes and automating is understandable. Databases have never been more important because data has never been more important. Every disaster nightmare a DBA, compliance officer, or PR team can imagine is wrapped around ensuring the database is safe.

In this talk, Kristyl Gomes and Robert Reeves will demonstrate why implementing standardization and automation is necessary to achieve what every team wants—speed and stability, with control.

1. Install Liquibase on our local system. Simply visit https://github.com/liquibase/liquibase/releases and get the latest release. 

	Note: Remember to download the PostgreSQL JDBC Driver and copy it to your liquibase/lib directory. Here's the latest from Maven Central: https://repo1.maven.org/maven2/org/postgresql/postgresql/42.2.19/postgresql-42.2.19.jar

2. Update your System Path to include your Liquibase installation.

3. Start with a Docker PostgreSQL database. 

	`docker run --name mypostgres -p 5432:5432 -e POSTGRES_PASSWORD=secret postgres`

4. Using your favorite SQL execution tool, connect to the Docker container and execute `init.sql`. This will create some simple tables and data. This will help us show how to use Liquibase against an existing database.

5. Configure Liquibase with a `liquibase.properties` file. Place this in your project directory. 

```
driver= org.postgresql.Driver
classpath= lib/postgresql-42.2.19.jar
url= jdbc:postgresql://localhost:5432/postgres
username= postgres
password= secret
changeLogFile= changelog.xml
```

5. Run `liquibase generatechangelog` to reverse engineer your database schema. This will create a `changelog.xml`.

6. Shutdown the Docker database and recreate it. 

	`docker stop mypostgres`

	`docker rm mypostgres`

	`docker run --name mypostgres -p 5432:5432 -e POSTGRES_PASSWORD=secret postgres`

7. Run `liquibase update`. Connect using your favorite SQL execution tool and examine the tables created.

8. Run `liquibase history` and review the status. 

9. Celebrate your freedom from the tyranny of manual database schema change!