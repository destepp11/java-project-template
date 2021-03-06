# Automated Builds/Deployments for Oracle databases.

# Table of Contents
	* Purpose
	* Benefits of Automated Builds/Deployments
	* The `.git` Repository Structure
	* Understanding Gradle and the `gradle.build` Script
	* Understanding Gradle `.properties` Files.
	* Understanding FlywayDB for Database Migrations
	* Understanding TeamCity Continous Integration
	* Using GitFlow

## Purpose
The following document outlines the Automated builds and deployments for Oracle databases. Though the documentation is specifically in reference to Oracle the same concepts are applicable to SQL Server, DB2, MySQL, MariaDB, PostgreSQL, Redshift, CockroachDB, and others. For a complete list of supported database refer [Flyway's documentation](https://flywaydb.org/documentation/).

## Benefits of Automated Builds/Deployments
    * Pratical
        Automated deployments are practical because they are configurable for the needs of the environment. For instance, to deploy changes to development we can use a .git commit trigger to start the build immediately. This doesn't require any involvement from management or database administrators to perform. Alternatively, for more gated environments like production deployments can only be triggered by the personelle with the appropriate access.
    * Safety
        Automated deployments are safe because executution of deployment task can be limited to partical roles or users. Also there is no human invention in the process which mitigates risk of human error. 
    * Repeatable
        Automated deployments are repeatable processes. The same deployment process can be executed in multiple environments ensuring what was executed in test is executed in production. 
    * Predictable
		Automated deployments are predictable because they are repeatable. When a production deployment is ran we know what to expect because the same script was ran in test. We can determine the sucess or failure of build/deployment because of this.

## The `.git` Repository Structure

In the root directory of the `.git` repository there are several scripts which are essential to the build and deployment process.

	### `src/*`
		This the the source directory of database objects and migration scripts. This is where developers will be placing their code.
	### `build.gradle`
		This file is the build script that needs to be executed to install the database/or application objects.
	### `config.groovy`
		This is a configuration file which contains the mapping of build parameters to Gradle `.properties` files. 
	### `dev.properties`
		This is a properties file where we declare environment specific parameters we need to pass into our `build.gradle` script.


## Understanding Gradle and the `build.gradle` Script

In short, Gradle is a build tool which allows developers to write build scripts in a domain specific language (DSL). In our implementation, we are using language features Java and Groovy. Techinically, any language that can run on a JVM can be used in a Gradle script, for example `Kotlin` or  `Ruby`.

Let's take a closer look at our implementation of `build.gradle` but we will be skipping some of the implementation details for the sake of time. First off, we need to include the FlywayDB gradle plugin:

	
	plugins {
    	id "java"
 	    id "org.flywaydb.flyway" version "5.0.6"
	}
	
Secondly, we need to include our dependencies for the Oracle jdbc driver, the Authorize.Net SDK, the Oracle payments library (included in `libs/payments.jar`), and also declare our unit testing framework:

    dependencies {
        compile("com.oracle:ojdbc6:11.2.0.4")
        compile group: 'net.authorize', name: 'anet-java-sdk', version: '1.9.5'
        compile fileTree(dir: 'libs', include: '*.jar')
        testCompile group: 'junit', name: 'junit', version: '4.12'
    }

We also require a way to load configuration and properties for certain environments: 

    loadConfiguration()

    def loadConfiguration() {
        def environment = project.hasProperty('env') ? project.env : 'dev' // Check the build command for the flag `-Penv` if it doesn't exist the environment will default to `dev`
        project.ext.set 'environment', environment
        println "Environment is set to $environment"

        def configFile = file('config.groovy') // Declare the configFile as `config.groovy`
        def config = new ConfigSlurper(environment).parse(configFile.toURI().toURL()) //Look for the value of the `-Penv` flag in the `config.groovy` file and save to the `config` variable.
        project.ext.set 'config', config //Set the project to use the `config`
    }

    def props = new Properties()
    file("$config.instanceProperties").withInputStream { props.load(it) } //Load the properties to our project's context


Finally, we set our flyway properties. More on this later:

    flyway {
        url = props.getProperty("flyway.url")
        driver = props.getProperty("flyway.driver")
        user = props.getProperty("flyway.user")
        password = props.getProperty("flyway.password")
        table = props.getProperty("flyway.table")
        baselineOnMigrate = props.getProperty("flyway.baselineOnMigrate")
    }



## Understanding Gradle `.properties` Files.

In our various properties files we can specify anything specfic for an target environment we would like. Currently we are only focused on the `flyway` properties, but I've include properties that are related to Oracle EBS and the Linux Server. Using properties files reduces the complexity of the build script because we abstract the implementation from the configuration. Any concerns about keeping this values in properties files can be mitigated by either referencing environment variables which are setup on the TeamCity build server, or can be passed in manually with build parameters.

`dev.properties`
	
	flyway.url = *******
	flyway.driver = oracle.jdbc.driver.OracleDriver
	flyway.user = apps
	flyway.password = apps
	flyway.table = IBY_schema_version
	flyway.baselineOnMigrate = true
	ebs.applmgr.user = *******
	ebs.applmgr.password = *******
	ebs.adopUser = *******
	ebs.adopPassword = *******
	ebs.webLogicPassword = *******
	linux.server.internal = *******
	linux.server.external = *******

`sandbox.properties`

	flyway.url = ***********
	flyway.driver = ***********
	flyway.user = ***********
	flyway.password = ***********
	flyway.table = IBY_schema_version
	flyway.baselineOnMigrate = true
	ebs.applmgr.user = ***********
	ebs.applmgr.password = ***********
	ebs.adopPassword = ***********
	ebs.webLogicPassword = ***********
	linux.server.internal = ***********
	linux.server.external = ***********

## Understanding FlywayDB for Database Migrations

[FlywayDB](https://flywaydb.org) is an excellent tool for executing database migrations. We will be using a FlywayDB plugin in conjuntion with our Gradle script.

FlwayDB takes several different parameters and an exhaustive list can be found [here](https://flywaydb.org/documentation/envvars).

We will be using the following parameters:

| Parameter                 | Description   |
| ------------------------- | -------------:|
| flyway.url                | The jdbc url to use to connect to the database. |
| flyway.driver             | The fully qualified classname of the JDBC driver to use to connect to the database. |
| flyway.user               | The user to use to connect to the database. |
| flyway.password           | The password to use to connect to the database. |
| flyway.table              | The name of Flyway's schema history table. By default (single-schema mode) the schema history table is placed in the default schema for the connection provided by the datasource. When the flyway.schemas property is set (multi-schema mode), the schema history table is placed in the first schema of the list. |
| flyway.baselineOnMigrate  | Whether to automatically call baseline when migrate is executed against a non-empty schema with no metadata table. This schema will then be baselined with the baselineVersion before executing the migrations. Only migrations above baselineVersion will then be applied. This is useful for initial Flyway production deployments on projects with an existing DB. Be careful when enabling this as it removes the safety net that ensures Flyway does not migrate the wrong database in case of a configuration mistake! |

When FlywayDB executes migration scripts it will do so in order. It's important for the developer to write their migration scripts in the correct order and with the correct nameing convention in order for FlywayDB to execute them. Migration scripts are to be stored in `src/main/resources/db/migrations/`. This is the default directory where FlywayDB expects migrations scripts. Migration scripts need to adhere to the following naming convention: `V1.1__Create_Table_HelloWorld.sql`
Flyway will create a new table to track the history of migrations. This is defined by the `flyway.table` parameter.


## Understanding TeamCity Continuous Integration

Our TeamCity will be responsible for the execution of the builds. We will create a new TeamCity project and define our VCS root as our repository. We will setup multiple build configurations for each environment and define the gradle tasks to be executed when a build is triggered. We we also setup build triggers as appropriate for each environment.

## What is Needed to Make Automated Builds and Deployments a Reality

* Team members should not get hung up on the edge cases. This solves 99% of our database activities. The 1% can be handled later.
* Understanding that Oracle EBS is nothing but software and it isn't scary.
* Understanding of our build and deployment process by all team members.
* Create a new Oracle EBS repo for these deployments and archieve the current one. We will not be migrating files from the current repo. Instead we will start a new baseline. The reason being because the combined object repo is not an accurate representation of the Oracle instance. We can't build the instance from source code now so no need to move possibly inaccurate information.
* All teams to follow GitFlow and submit pull requests to the release branch when ready for migration.
* Pull requests will serve as the approval for migration from this point forward.
* Release Manager to create release branches and merge into branches for the target environments to trigger the builds.
* Properties for protected environments to be provided to the TeamCity build agents.
* TeamCity project and build configuration to be created.
* Expand the build script to handle application side deployments for instance Java applications. Currently the iStore repo's build script can migrate automatically to development and sandbox but should be refactored to reduce complexity and leverage the `.properties` files.