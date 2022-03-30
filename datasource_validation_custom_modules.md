# Enable configuration of custom datasource validation modules

IronJacamar project (which is an underlying project on top of which WildFly's datasources substem is implemented) allows to configure classes that are used to validate whether datasource connection is working fine (*org.jboss.jca.adapters.jdbc.spi.ValidConnectionChecker*), or, if the exception is being thrown during some operation using a connection, check whether it is being stale (*org.jboss.jca.adapters.jdbc.spi.StaleConnectionChecker*) and whether this exception is being fatal (*org.jboss.jca.adapters.jdbc.spi.ExceptionSorter*).

Those can be configured in WildFly's connector subsystem in the following way:

```
<subsystem xmlns="urn:jboss:domain:datasources:7.0">
    <datasources>
        <datasource jndi-name="java:jboss/datasources/ExampleDS" pool-name="ExampleDS"
                    use-java-context="true">
            <connection-url>jdbc:h2:mem:test;DB_CLOSE_DELAY=-1</connection-url>
            <driver>h2</driver>
            <security>
                <user-name>sa</user-name>
                <password>sa</password>
            </security>
            <validation>
                <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.novendor.DBC4ValidConnectionChecker"/>
                <stale-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.novendor.AlwaysStaleConnectionChecker"/>
                <exception-sorter class-name="org.jboss.jca.adapters.jdbc.extensions.novendor.AlwaysExceptionSorter"/>	
            </validation>
        </datasource>
        <drivers>
            <driver name="h2" module="com.h2database.h2">
                <xa-datasource-class>org.h2.jdbcx.JdbcDataSource</xa-datasource-class>
                <datasource-class>org.h2.jdbcx.JdbcDataSource</datasource-class>
            </driver>
        </drivers>
    </datasources>
</subsystem>
```

Please note that until recently validation classes needed to be a part of IronJacamar code as they were loaded from IronJacamar's JDBC library. Effectively, this made writing custom validation classes impossible.

Since WildFly 26.1 you are able to configure custom module and, as a result, write your own implementation of a validation. I'm going to present how to do this by implementing custom valid connection checker class. (I have uploaded the code used in this blog to [GitHub](https://github.com/tadamski/wildfly-datasource-lazy-validator), so that you can test it in your own WildFly server.) 

First thing that we need to to is create a module project. Let's use Maven for that:

```
(...)
<groupId>org.wildfly.examples.datasources</groupId>
<artifactId>lazy-validation-module</artifactId>
<name>lazy-validation-module</name>
<packaging>jar</packaging>
<version>1.0.0</version>

<properties>
    <version.org.jboss.ironjacamar>1.5.5.Final</version.org.jboss.ironjacamar>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.jboss.ironjacamar</groupId>
            <artifactId>ironjacamar-jdbc</artifactId>
            <version>${version.org.jboss.ironjacamar}</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.jboss.ironjacamar</groupId>
        <artifactId>ironjacamar-jdbc</artifactId>
    </dependency>
</dependencies>
(...)
```

Our project is called *lazy-validation-module* and depends on *org.jboss.ironjacamar:ironjacamar-jdbc* which cointains *org.jboss.jca.adapters.jdbc.spi.ValidConnectionChecker* interface.

Let's now look at the validation checker implementation:

```
package org.wildfly.examples.datasources;

import org.jboss.jca.adapters.jdbc.spi.ValidConnectionChecker;

import java.sql.Connection;
import java.sql.SQLException;
import java.util.logging.Logger;

public class LazyValidationChecker implements ValidConnectionChecker {
    Logger logger = Logger.getLogger(LazyValidationChecker.class.getName());

    public SQLException isValidConnection(Connection c) {
        logger.info("Everything is fine...");
        return null;
    }
}
```

As you are able to see, our checker does nothing and always concludes that the connection is valid. It logs the information about validation being performed though so that we will be able to verify whether it has been executed.

Finally, we have to create a *module.xml* file:

```
<?xml version="1.0" encoding="UTF-8"?>

<module name="org.wildfly.examples.datasources.lazyvalidation"  xmlns="urn:jboss:module:1.9">

    <resources>
        <resource-root path="lazy-validation-module-1.0.0.jar"/>
    </resources>
    
    <dependencies>
        <module name="javax.sql.api"/>
        <module name="sun.jdk"/>
        <module name="javax.orb.api"/>
        <module name="java.logging"/>
        <module name="org.jboss.ironjacamar.jdbcadapters"/>
    </dependencies>    
</module>
```

The module is called *org.wildfly.examples.datasources.lazyvalidation* and cointains *org.jboss.ironjacamar.jdbcadapters* dependency plus other dependencies needed for it to work.

```
[tomek@localhost main]$ export WIDFLY_HOME=... #your WildFly directory
[tomek@localhost main]$ export VALIDATOR_HOME=... #your lazy validator module directory
[tomek@localhost main]$ cd ${VALIDATOR_HOME}
[tomek@localhost main]$ mvn clean install 
[tomek@localhost main]$ cd ${WIDFLY_HOME}
[tomek@localhost main]$ mkdir -p modules/org/wildfly/examples/datasources/lazyvalidation/main
[tomek@localhost main]$ cp ${VALIDATOR_HOME}/target/lazy-validation-module-1.0.0.jar modules/org/wildfly/examples/datasources/lazyvalidation/main
[tomek@localhost main]$ cp ${VALIDATOR_HOME}/module.xml modules/org/wildfly/examples/datasources/lazyvalidation/main
```

Finally, we have to edit *${WILDFLY_HOME}/standalone/configuration/standalone.xml* file to let WildFly know that we are going to use our customer validator:

```
(...)
<subsystem xmlns="urn:jboss:domain:datasources:7.0">
    <datasources>
        <datasource jndi-name="java:jboss/datasources/ExampleDS" pool-name="ExampleDS" enabled="true" use-java-context="true" statistics-enabled="${wildfly.datasources.statistics-enabled:${wildfly.statistics-enabled:false}}">
        <connection-url>jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE</connection-url>
        <driver>h2</driver>
        <security>
            <user-name>sa</user-name>
            <password>sa</password>
		</security>
		<validation>
			<valid-connection-checker class-name="org.wildfly.examples.datasources.LazyValidationChecker" module="org.wildfly.examples.datasources.lazyvalidation"/>
		</validation>
    </datasource>
    <drivers>
        <driver name="h2" module="com.h2database.h2">
            <xa-datasource-class>org.h2.jdbcx.JdbcDataSource</xa-datasource-class>
        </driver>
    </drivers>
    </datasources>
</subsystem>
```

Now we are ready to start WildFly:

```
[tomek@localhost main]$ cd ${WIDFLY_HOME}
[tomek@localhost main]$ bin/standalone.sh
```

Let's verify that the validator is indeed working. Let's login to WildFly's CLI in different terminal window and perform the following commands:

```
[tomek@localhost main]$ cd ${WIDFLY_HOME}
[tomek@localhost main]$ bin/jboss-cli.sh
[disconnected /] connect
[standalone@localhost:9990 /] cd subsystem=datasources/data-source=ExampleDS
[standalone@localhost:9990 data-source=ExampleDS] :test-connection-in-pool
[standalone@localhost:9990 data-source=ExampleDS] :flush-invalid-connection-in-pool
```

By executing the commands above we are connecting to CLI and going to *ExampleDS* directory. We are executing *test-connection-in-pool* command first to establish a connection so that our validator has something it can work on. Later we are performing *flush-invalid-connection-in-pool*, which performs validation on each connection in an attempt to flush invalid ones. Nevertheless, after using this trick we are able to see in server log that the *LazyValidatorChecker's* method has been run:

```
(...)
23:33:45,963 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0060: Http management interface listening on http://127.0.0.1:9990/management
23:33:45,963 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0051: Admin console listening on http://127.0.0.1:9990
23:38:24,659 INFO  [org.wildfly.examples.datasources.LazyValidationChecker] (management-handler-thread - 1) Everything is fine..
```


 


