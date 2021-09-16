# Alfresco crone services
> The project is here: <https://gitlab.com/fedorow/alfresco-cron-services>

This docker container can run any crone jobes in Alfresco Docker or Docker Compose doployments. It run scrips by cron based scedule. By default it clean deleted contentstore and remove old log files. Any custom shell script can be executed by cron inside the container. See, image customisation section. 

Container must have mapped volumes to the alf_data/contentstore.deleted directory and to the alfresco, share and postgers log files directories. Sciped folders will not be procesed. 

### Deleted contentstore cleaner
Alfresco have multistage process of deleting documents. In fact, Alfresco never remove bin files from storage.  You can read more about Alfresco deletion process at https://blog.dbi-services.com/understand-the-lifecycle-of-alfresco-nodes/. At the end of it's process bin files of deleted nodes stay in the contentstore.deleted folder. It should be deleted by admin, to free starage space.

The default age of bin files, older of which must be deleted:
```
DELETED_DAYS=30
```
Empty folders will be deleted after $DELETED_DAYS + 30 days.

### Logs cleaner
Logs files of tomcat and postgress can takes a lot's of storage. The default age of logs files, older of which must be deleted:
```
ALFRESCO_LOGS_DAYS=7 
SHARE_LOGS_DAYS=14
POSTGRES_LOGS_DAYS=14
```
Container add INFO records into alfresco.log file. To avoid it set enviroment variable LOGS to false.

### Docker usage
To download image:
```
docker pull fedorow/alfresco-cron-services:latest
```
To run container with default properties:
```
sudo docker run --name cron-services -d \
  -v <your_alf_data_path>/contentstore.deleted:/contentstore.deleted \
  -v <your_alfresco_logs_path>:/logs/alfresco \
  -v <your_share_logs_path>:/logs/share \
  -v <your_postgres_logs_path>:/logs/postgres \
  fedorow/alfresco-cron-services:latest
```
### Docker Compose usage
1. Add cron-service to the project and map volumes to the propper folders:
```
services:
  alfresco:
  ...

  cron-services:
    image: fedorow/alfresco-cron-services:latest
    volumes:
      - <your_alf_data>/contentstore.deleted:/contentstore.deleted
      - <your_alfresco_logs_path>:/logs/alfresco
      - <your_share_logs_path>:/logs/share
      - <your_postgres_logs_path>:/logs/postgres
```
2. To change default properties add optional enviromwnt section. For example, change age of files, stop updating alfresco.log file and change time zone of docker container to the system time of server:
```
  cron-services:
    image: fedorow/alfresco-cron-services:latest
    environment:
      DELETED_DAYS=90
      ALFRESCO_LOGS_DAYS = 10
      SHARE_LOGS_DAYS=30
      POSTGRES_LOGS_DAYS=30
      LOGS=false
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - <your_alf_data>/contentstore.deleted:/contentstore.deleted
      - <your_alfresco_logs_path>:/logs/alfresco
      - <your_share_logs_path>:/logs/share
      - <your_postgres_logs_path>:/logs/postgres

```

### Image customisation

To add custom scripts to cron jobes make a cunsom image. For example, to execute myScript.sh every 10 minutes.

Dockerfile:
```
FROM fedorow/alfresco-cron-services:latest

COPY myScript.sh /root/myScript.sh
COPY mycrontab.txt /root/mycrontab.txt

RUN /usr/bin/crontab /root/mycrontab.txt
```
mycrontab.txt:
```
*/10 * * * * /root/myScript.sh
# empty line
```
myScript.sh:
```
#!/bin/ash
echo "This script does it every 10 minutes"
```
Script will be executes by root. The annotation of script is sh.
