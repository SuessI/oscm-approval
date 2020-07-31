# oscm-approval
OSCM Approval Tool (WIP)


## Build ##
Run maven on the parent pom
mvn clean package

Results in <projectroot>/target/approval-impl-0.0.2-SNAPSHOT.war 

## Prepare ##
Specify where OSCM installation folder /docker is contained 
```
export WORKSPACE=<YOUR OSCM HOME>
```
Copy the extracted content into /docker
```
cp -R . $WORKSPACE/docker
cd $WORKSPACE/docker
```

## Init DB
Run following commands to create and init the approval db 
```
docker run -it --name seconddb --rm --network docker_default -v $WORKSPACE/docker/init:/initapproval artifactory.intern.est.fujitsu.com:5003/oscmdocker/oscm-db bash
export PGPASSWORD=secret
psql -h oscm-db -p 5432 -U postgres -f /initapproval/approval_init.sql
psql -h oscm-db -p 5432 -U approvaluser -W -d approvaldb -f /initapproval/upd_postgresql_01_01_01.sql
```

## 
Copy the war file and following resources into the app container
```
docker cp tomee.xml oscm-app:/opt/apache-tomee/conf/
docker cp tomee_template.xml oscm-app:/opt/apache-tomee/conf/
docker cp tomcat-users.xml oscm-app:/opt/apache-tomee/conf/

docker cp approval-impl-0.0.2-SNAPSHOT.war  oscm-app:/opt/apache-tomee/webapps/approval.war
docker logs -f oscm-app
```

## Check Deployment
See logs, e.g. ApprovalNotificationService is well deployed, e.g.
```
31-Jul-2020 15:18:55.211 INFO [localhost-startStop-1] org.apache.openejb.server.webservices.WsService.afterApplicationCreated Webservice(wsdl=http://localhost:8880/approval/ApprovalNotificationService, qname={http://oscm.org/xsd}ApprovalNotificationService) --> Pojo(id=localhost.approval.org.oscm.app.approval.triggers.ApprovalNotificationService)
31-Jul-2020 15:18:55.218 INFO [localhost-startStop-1] sun.reflect.DelegatingMethodAccessorImpl.invoke Deployment of web application archive [/opt/apache-tomee/webapps/approval.war] has finished in [7,287] ms
```

## Test it
Login to `https://<yourhost>:8881/approval/`
	user: approval
	password: approval
OK
```
https://<yourhost>:8881/approval/ApprovalNotificationService?wsdl
```

### Now insert DB configsettings ### 
In order to configure the Sample Supplier 'supplier' of '959c9bf7' as user supplier the service to be triggered
```
docker run -it --name seconddb --rm --network docker_default -v $WORKSPACE/docker/init:/initapproval artifactory.intern.est.fujitsu.com:5003/oscmdocker/oscm-db bash
export PGPASSWORD=secret
psql -h oscm-db -p 5432 -U postgres -d bssapp -c "INSERT INTO bssappuser.configurationsetting (controllerid, settingkey, settingvalue) VALUES ('ess.vmware', 'USERID_959c9bf7', 'supplier');"
psql -h oscm-db -p 5432 -U postgres -d bssapp -c "INSERT INTO bssappuser.configurationsetting (controllerid, settingkey, settingvalue) VALUES ('ess.vmware', 'USERPWD_959c9bf7', '_crypt:supplier');"
```

### Now insert your APPROVAL_URL with your FQDN ### 
(!!! edit at the end of following command !!!) 
```
psql -h oscm-db -p 5432 -U postgres -d bssapp -c "INSERT INTO bssappuser.configurationsetting (controllerid, settingkey, settingvalue) VALUES ('ess.vmware', 'APPROVAL_URL', 'https://<yourhost>:8881/approval/');"
docker restart oscm-app
```

### Prepare Trigger
1. Login to OSCM Marketplace as Supplier 
2. Goto Account > Processes
3. Create New Trigger Process
Display as: Approval Trigger

NOTE: Due to a current bug https://github.com/servicecatalog/oscm/issues/1041 trigger definitions can be changed, so take care (workaround is to change it directly in the database, eg. pgAdmin)


	Type: Subscribe to Service
	Target type: Web Service
	Target URL: http://oscm-app:8880/approval/ApprovalNotificationService?wsdl
	Suspend: Yes (check the checkbox!)

### Subscribe a service
1. If not done, deploy the Sample Controller in the oscm-app container and create a respective service.
2. Subscribe the service -> Subscription is suspended with trigger message 
3. Login again as manager "approval" to `https://<yourhost>:8881/approval/` 
