# drools-tomcat

### Download Tomcat

Download Tomcat zip and deploy it.

- [Apache Tomcat](https://tomcat.apache.org/download-90.cgi)

### Download war

Firstly, make sure you download workbench and KIE Server for the container you target. Apache Tomcat, the following are required:

- [Kie Server](https://download.jboss.org/drools/release/6.5.0.Final/kie-server-distribution-6.5.0.Final.zip)
- [Kie Workbench](https://download.jboss.org/drools/release/6.5.0.Final/kie-drools-wb-6.5.0.Final-tomcat7.war)

### Deploy applications

Copy downloaded files into TOMCAT_HOME/webapps, while copying rename them to simplify the context paths that will be used on application server:

- Rename kie-drools-wb-6.5.0.Final-tomcat7.war to kie-wb.war
- Rename kie-server-6.5.0.Final-webc.war to kie-server.war

### Configure server

- Copy libraries in libs folder into TOMCAT_HOME/lib
- Create Bitronix configuration files ‘btm-config.properties’ to enable JTA transaction manager under TOMCAT_HOME/conf with following content:
  - [btm-config.properties](configs/btm-config.properties)
- Create file ‘resources.properties’ under TOMCAT_HOME/conf with following content

  - [resources.properties](configs/resources.properties)

- Configure users. Create following users in tomcat-users.xml under TOMCAT_HOME/conf:

  ```
  <tomcat-users>
      <role rolename=”admin”/>
      <role rolename=”kie-server”/>
      <role rolename=”manager-gui”/>
      <role rolename=”manager”/>
      <role rolename=”analyst”/>
      <role rolename=”user”/>
      <role rolename="rest-all"/>
      <role rolename="developer"/>

      <user username=”admin” password=”pass” roles=”admin,manager,manager-gui”/>
      <user username=”kieserver” password=”pass” roles=”kie-server,rest-all”/>
      <user username=”developer” password=”dev” roles=”user,developer,analyst”/>
  </tomcat-users>
  ```

- Configure system properties. Create a file setenv.bat under TOMCAT_HOME/bin. Configure following system properties in file setenv.bat for Windows(setenv.sh for Linux):

```
export JAVA_HOME=/opt/jdk/11.0_19164
export CATALINA_HOME=/home/rockgway/apache-tomcat/apache-tomcat-9.0.93
export CATALINA_PID="$CATALINA_HOME/temp/tomcat.pid"

export CATALINA_OPTS="-Xmx512M -XX:MaxPermSize=512m \
-Dbtm.root=$CATALINA_HOME \
-Dbitronix.tm.configuration=$CATALINA_HOME/conf/btm-config.properties \
-Dorg.jbpm.cdi.bm=java:comp/env/BeanManager \
-Dbitronix.tm.configuration=$CATALINA_HOME/conf/btm-config.properties \
-Djbpm.tsr.jndi.lookup=java:comp/env/TransactionSynchronizationRegistry \
-Djava.security.auth.login.config=$CATALINA_HOME/webapps/kie-wb/WEB-INF/classes/login.config \
-Dorg.kie.server.persistence.ds=java:comp/env/jdbc/jbpm \
-Dorg.kie.server.persistence.dialect=org.hibernate.dialect.H2Dialect \
-Dorg.kie.server.persistence.tm=org.hibernate.service.jta.platform.internal.BitronixJtaPlatform \
-Dorg.kie.server.id=tomcat-kieserver \
-Dorg.kie.demo=false \
-Dorg.kie.server.location=http://localhost:8080/kie-server/services/rest/server \
-Dorg.kie.server.controller=http://localhost:8080/kie-wb/rest/controller"

```

- Make sure your TOMCAT_HOME/conf/context.xml looks like below:

```
<?xml version=”1.0" encoding=”UTF-8"?>
<Context reloadable=”true”>
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <WatchedResource>WEB-INF/tomcat-web.xml</WatchedResource>
    <WatchedResource>${catalina.base}/conf/web.xml</WatchedResource>
    <Transaction factory=”bitronix.tm.BitronixUserTransactionObjectFactory”/>
    <CookieProcessor className=”org.apache.tomcat.util.http.LegacyCookieProcessor” />
</Context>
```

- Add the following listener in TOMCAT_HOME/conf/server.xml under `<Server>` tag:

```
<Listener className=”bitronix.tm.integration.tomcat55.BTMLifecycleListener”/>
```

- Add the following valve (to include jar jacc-1.0) in TOMCAT_HOME/conf/server.xml under `<Host>` tag:

```
<Valve className=”org.kie.integration.tomcat.JACCValve” />
```

### Changing database persistence settings

Since by default persistence uses just in memory data base (H2) it is good enough for first tryouts or demos but not for real usage. So to be able to change persistence settings following needs to be done:

- Change file ‘resources.properties’ under TOMCAT_HOME/conf with following content:

```
resource.ds1.className=bitronix.tm.resource.jdbc.lrc.LrcXADataSource
resource.ds1.uniqueName=jdbc/jbpm
resource.ds1.minPoolSize=10
resource.ds1.maxPoolSize=20
resource.ds1.driverProperties.driverClassName=com.mysql.jdbc.Driver
resource.ds1.driverProperties.user=admin1
resource.ds1.driverProperties.password=pass
resource.ds1.driverProperties.url=jdbc:mysql://localhost:3306/jbpm?useSSL=false
resource.ds1.allowLocalTransactions=true
```

- Copy MySQL JDBC driver into TOMCAT_HOME/lib otherwise it won’t provide proper connection handling.

- Next we will modify persistence.xml that resides inside workbench war file.
  > Extract the kie-wb.war file into directory with same name and in same location (TOMCAT_HOME/webapps). Navigate to kie-wb.war/WEB-INF/classes/META-INF. Edit persistence.xml file and change following elements:

1.  jta-data-source to point to the newly created data source (JNDI name) for your data base.
    The code sample is:

```
<persistence-unit name=”org.jbpm.domain” transaction-type=”JTA”>
<provider>org.hibernate.ejb.HibernatePersistence</provider>
<jta-data-source>java:comp/env/jdbc/jbpm</jta-data-source>
```

Here, jdbc/jbpm is JNDI name for your data base. You can change it as per your requirement.

2. hibernate.dialect to hibernate supported dialect name for your database. Under `<properties>` tag, change the following property according to your DB Dialect:

```
<property name=”hibernate.dialect” value=”org.hibernate.dialect.MySQL5Dialect”/>
```

- Set or modify (as data source is already defined there) following system properties in setenv.bat script inside TOMCAT_HOME/bin:

```
-Dorg.kie.server.persistence.ds=java:comp/env/jdbc/jbpm
-Dorg.kie.server.persistence.dialect=org.hibernate.dialect.MySQL5Dialect
```

### Run & Stop Server

```
./startup.sh

./shutdown.sh

```
