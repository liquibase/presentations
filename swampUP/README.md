

# swampUP 2020
# IT’S TIME TO AUTOMATE YOUR DATABASE SCHEMA CHANGES

Let’s remember why we use Artifactory in the first place! We need a single source of truth. Of course, we have that with our source code control but that’s not the best place to put our applications that are ready for release. Artifactory provides a mechanism for anyone in the organization to request the latest released version of an application and also include any dependencies required by the new application version.

However, most of you are not including the database schema changes. If you’re including your database as part of your architecture, why aren’t you including it as an artifact? This oversight is causing serious problems.

Cramming SQL scripts into Artifactory isn’t the answer. It’s time to automate your database schema change process.

Anyone can do this. I’ll demonstrate how you can use an open source tool called Liquibase to make database schema available for test environments. If you have a lot of database migrations to develop, your integration test environments should automatically be in sync with the defined migrations. The most efficient way to handle this is to let Jenkins push the Liquibase defined ChangeLog (whether XML, YAML, JSON, or annotated SQL) to your internal Artifactory. Thus, your application and dependent database changes will be part of a distinct, atomic release.

1. Start with a Docker PostgreSQL database using the Sakila database schema. This will represent our existing database at which we are currently throwing SQL scripts. Boo!

You can use Sakila from our friends at jOOQ; they have it for DB2, MySQL, Oracle, Posgres, SQL Server, and SQL Lite. You can find the one we used in the Sakila directory in this repository.

`docker run -p 5432:5432 -e POSTGRES_PASSWORD=secret postgres`

Connect with DBeaver or your favorite SQL execution tool. Then execute https://github.com/jOOQ/jOOQ/blob/master/jOOQ-examples/Sakila/postgres-sakila-db/postgres-sakila-schema.sql. You can verify that the objects were created by looking in DBeaver.

2. Create a changelog from your new database. I created a liquibase.properties file. You can find it in the Liquibase folder in this repository.
	
I have Liquibase 3.9.0 on my system path because I'm awesome. Thus, from the Liquibase folder I executed the following:

`liquibase generatechangelog`



4. Now this is where we pull a Robert Frost and notice that there are roads in the yellow wood. 


	a. Imagine this database is a production database and you are attempting to implement true CI/CD for the database. Thus, you are going to assert that all of the changes in the changelog.xml have been executed on the server. Going forward, you will simply add a new changeset to the end of the changelog.xml, execute `liquibase update`, and watch as your databse is updated via automation.
	
	To do this, you will simply execute `liquibase changeLogSync`, which creates that DATBASECHANGELOG table and stores all the changesets there. This is used on subsequent updates so Liquibase knows which changes have been run and, thus, Liquibase will only run the updates necessary.
	
	
	b. However, we really need to test our changelog.xml before we do that. After all, there could be issues with the changelog.xml and it would be bad form to check in code that we don't know if it works. (Spoiler: there's a problem)
	
		i. Start a new PosgreSQL database. Or you can shutdown the running one and restart it. That's what I'm going to do.

		`docker stop <container name>`
		`docker run -p 5432:5432 -e POSTGRES_PASSWORD=secret postgres`
	
		A side note about Liquibase: Liquibase was started in 2006 by Nathan Voxland (@nvoxland) for the sole purpose of allowing him to deploy database code to both Oracle and SQL Server without changing the underlying code. That means Liquibase is platform agnostic. So, if you tell it to create a table, it can do that in MySQL, Hibernate, Postgres, any supported database.
		
		But, that's where we run into issues somethimes. The Sakila schema has platform specifc objects that don't mean much on other platforms. So, you'll have to include the SQL.


		To see this in action, execute `liquibase update` and you will see an error on creating the table 'film'. This is because table 'film' has two columns that use a RULE and custom data TYPE. So, we need to add the following SQL statements before that table is created:
		
		`
		<changeSet author="r2" id="34.1">
			<sql>CREATE DOMAIN year AS integer CONSTRAINT year_check CHECK (((VALUE &gt;= 1901) AND (VALUE &lt;= 2155)))</sql>
			<sql>ALTER DOMAIN public.year OWNER TO postgres</sql>
		</changeSet>

		<changeSet author="r2" id="34.2">
			<sql>CREATE TYPE mpaa_rating AS ENUM ('G', 'PG', 'PG-13', 'R', 'NC-17')</sql>
			<sql>ALTER TYPE public.mpaa_rating OWNER TO postgres</sql>
		</changeSet>
		`

		You can certainly give any id you want, but I chose this because the 'film' table changeset is, in my changelog, "<epoch time>-34"
		
		Our next fix will be the "YEAR (10)" datatype on the release_year column in the film table. Simply change "YEAR (10)" to "YEAR".
		
		Our next step will be including functions that are dependent from some views in the schema. Of course, this functionality  is part of Liquibase Pro.
		
		<changeSet author="r2" id="51.1">
			<sql>CREATE FUNCTION _group_concat(text, text) RETURNS text
						AS $_$
					SELECT CASE
					  WHEN $2 IS NULL THEN $1
					  WHEN $1 IS NULL THEN $2
					  ELSE $1 || ', ' || $2
					END
					$_$
						LANGUAGE sql IMMUTABLE;
			</sql>
			<sql>ALTER FUNCTION public._group_concat(text, text) OWNER TO postgres;</sql>
		</changeSet>
		<changeSet author="r2" id="51.2">
			<sql>CREATE AGGREGATE group_concat(text) (
						SFUNC = _group_concat,
						STYPE = text
					);
			</sql>
			<sql>ALTER AGGREGATE public.group_concat(text) OWNER TO postgres;</sql>
		</changeSet>

		Of course, this file is included in the repo.

5. Tie it all together with Artifactory and Jenkins

	a. Upload the Changelog to Artifactory. I'm using Artifactory Cloud because I'm lazy and don't want to set this up myself.
	
	`curl -uadmin:APAGRiLWyZWEGUypoQJ6jHFMJ3c -T changelog.xml "https://liquibase.jfrog.io/artifactory/example-repo-local/changelog.xml"`
	
	b. Start a Jenkins Docker container. Again, lazy == efficient.
	
	`docker run -p 8080:8080 jenkins/jenkins:lts`
	
	c. Configure Liquibase on Jenkins instance. Note: You can skip this step if you just want to use the GitHub repos, but keeping it here for reference.
		
	`
	docker exec -it -u root <container> bash
	cd /tmp
	curl -L https://github.com/liquibase/liquibase/releases/download/v3.9.0/liquibase-3.9.0.tar.gz --output liquibase-3.9.0.tar.gz
	mkdir liquibase
	cd liquibase
	tar xvzf ../liquibase-3.9.0.tgz
	curl -L https://repo1.maven.org/maven2/org/postgresql/postgresql/42.2.9/postgresql-42.2.9.jar --output /tmp/liquibase/lib/postgresql-42.2.9.jar
	`
	
	c. Configure Jenkins
	
	Get the IP address of the Postgres instance so Liquibase on Jenkins can connect.
	
	`docker inspect <postgres container>`
	
	# configure jenkins freeform job
	
	git: https://github.com/liquibase/presentations.git
	shell: 
		chmod +x ./swampUP/Liquibase/liquibase-3.9.0/liquibase
		./swampUP/Liquibase/liquibase-3.9.0/liquibase --changeLogFile=./swampUP/Liquibase/changelog.xml --username=postgres --password=secret --url=jdbc:postgresql://172.17.0.3:5432/postgres update
	

