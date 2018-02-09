# Automated Builds/Deployments for Oracle databases.

# Purpose
The following document outlines the Automated builds and deployments for Oracle databases. Though the documentation is specifically in reference to Oracle the same concepts are applicable to SQL Server, DB2, MySQL, MariaDB, PostgreSQL, Redshift, CockroachDB, and others. For a complete list of supported database refer [Flyway's documentation](https://flywaydb.org/documentation/).

# Table of Contents
    * Pratical
        Automated deployment are practical because they are configurable for the needs of the environment. For instance, to deploy changes to development we can use a .git commit trigger to start the build immediately. This doesn't require any involvement from management or database administrators to perform. Alternatively, for more gated environments like production deployments can only be triggered by the personelle with the approiate access.
    * Safety
    * Effecient
    * Repeatable
    * Predictable
    


