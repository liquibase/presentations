# DevOps World 2020
# Your CI/CD Fails at the Database and What You Can Do About It

Speed is everything right now. At the same time, a global pandemic has led companies to reduce headcount. Many organizations have already adopted automation in their software delivery process to help developers build and continuously deliver app code. But in most cases, the database has been left behind. What’s at stake? Everything. Databases have never been more important because data has never been more important. Without automation, CI/CD fails at the database. This session will teach you how you can step up to: 
	- Automate database schema chang
	- Automatically enforce standards 
	- Release the same way to Test and Prod
	- You can’t DBA yourself out of this
	
This session will help you learn key skills that will keep your organization competitive.

1. Start with a Docker PostgreSQL database. 

	`docker run -p 5432:5432 -e POSTGRES_PASSWORD=secret postgres`

2. Now, we need to get Jenkins up and running with the Liquibase Runner Plugin. (This will be republished soon after some final reviews, but for now...)

	`git clone https://github.com/jenkinsci/liquibase-runner-plugin.git`
	`cd liquibase-runner-plugin`
	`mvn hpi:run`
	
3. Next, we need to get Liquibase installed on our local system. Simply visit https://github.com/liquibase/liquibase/releases and get the latest release. We'll use 3.10.2 for this demo.

	Note: Remember to download the PostgreSQL JDBC Driver and copy it to your liquibase/lib directory. Here's the latest from Maven Central: https://repo1.maven.org/maven2/org/postgresql/postgresql/42.2.9/postgresql-42.2.9.jar
	
4. Now the fun part: Let's configurate Jenkins!

	Go to Manage Jenkins --> Global Tool Configuration --> Liquibase Installations and create a new Global Liquibase installation:
		Name: Liquibase 3.10.2
		Liquibase Installation Path: Path to Liquibase 3.10.2
		Select Save	
		
	Go to Jenkins --> <Your Project> --> Configure and make sure you have the following configurated.
	
	Source Code Management --> Git -> https://github.com/liquibase/presentations.git (no need for credentials)
		
	Add Build Step --> Liquibase: Update Database
		Change Log: ${WORKSPACE}/DevOpsWorld/liquibase/changelog.xml
		Database URL: jdbc:postgresql://localhost:5432/postgres (use `docker inspect <container>` to verify the IP of your container.)
		Credentials: postgres/secret
		Liquibase Installation: 3.10.2

Now build it! Don't forget to celebrate your freedome from the tyranny of manual database schema change!!!
