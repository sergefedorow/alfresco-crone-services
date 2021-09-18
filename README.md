# Alfresco crone services
> **The real project is here: <https://gitlab.com/fedorow/alfresco-cron-services>**


This docker container can run crone jobes in Alfresco Docker, Docker Compose or Kubernetes doployments. It run linux shell scrips by cron based scedule. Image have two preinstalled scripts. Every night it clean deleted contentstore and remove old log files. Custom shell scripts can be added inside the container (see [Image customisation](#image-customisation) section).

Cron-services Container must have mapped persistant volumes to the alf_data/contentstore.deleted directory, to the alfresco, share and postgers log files directories.

### Deleted contentstore cleaner
Alfresco have multistage process of deleting documents. In fact, Alfresco never remove bin files from storage.  Read more about Alfresco deletion process at the article **[Understand the Lifecycle of Alfresco Nodes](https://blog.dbi-services.com/understand-the-lifecycle-of-alfresco-nodes/)**. At the end of it's process bin files of deleted nodes stay in the contentstore.deleted folder. To free starage space it should be deleted.
Every 3:16AM contentstore cleaner remove old bin files and folders from contentstore.deleted folder.
The default age of bin files, older of which must be deleted:
```
DELETED_DAYS=30
```
Empty folders will be deleted after $DELETED_DAYS + 30 days.

### Logs cleaner
Logs files of tomcat and postgress can takes a lot's of storage. Cron-services delete alfresco,share and postgres logs older then aplyed by enviroment variables. By default:
```
ALFRESCO_LOGS_DAYS=7 
SHARE_LOGS_DAYS=14
POSTGRES_LOGS_DAYS=14
```
During execution, scripts add INFO records into alfresco.log file. To avoid it set enviroment variable LOGS to false.

### Time zone
By default cron-services container works and run cronjobes in UTC+0 time. To make it work in local time, time zone must be changed. For linux host machines easiest way is map /etc/localtime file to the container in read only mode.
```
-v /etc/localtime:/etc/localtime:ro
```

### Docker usage
To download image:
```
docker pull fedorow/alfresco-cron-services:latest
```
To run container with default properties:
```
sudo docker run --name cron-services -d \
  -v <path_to_alf_data>/contentstore.deleted:/contentstore.deleted \
  -v <path_to_alfresco_logs>:/logs/alfresco \
  -v <path_to_share_logs>:/logs/share \
  -v <path_to_postgres_logs>:/logs/postgres \
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

To add custom scripts to cron-services make a cunsom image. Scripts will be executed by root. The annotation of scripts must be sh.
For example, to execute myScript.sh every 10 minutes:

Dockerfile
```
FROM fedorow/alfresco-cron-services:latest

# Copy custom script to the container and make it executable
COPY myScript.sh /root/myScript.sh
RUN chmod +x /root/myScript.sh

# Update crontab by custom cron
RUN echo "*/10 * * * * /root/myScript.sh" >> /root/crontab.txt && \
    echo "" >> /root/crontab.txt
RUN /usr/bin/crontab /root/crontab.txt
```

myScript.sh
```
#!/bin/ash
echo "This script does it every 10 minutes"
```
