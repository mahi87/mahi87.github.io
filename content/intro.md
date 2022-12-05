Title: Restore data from bak-file for MS-SQL using Docker (WIP)
Date: 2022-12-05 10:20
Category: docker, mssql, how-to


> Do you want to restore data from a backup file and don't want to use mssql server?  
> **Creating docker image is the answer.** 

## Why I am ditching SSMS/mssql server studio

1. Full blown installation 
2. for fresh DB, it needs a complete clean up.
3. I just wanted to restore data and do not need to fire queries necessarily. 

## Build docker image

Open a text editor and copy below content into `Dockerfile` 

```Dockerfile

FROM mcr.microsoft.com/mssql/server:2017-latest

ENV ACCEPT_EULA=y
ENV MSSQL_SA_PASSWORD=YourPassword
ENV MSSQL_PID=Express

ARG PATH_TO_YOUR_BACKUP_FILE
ARG your_db_name=restored_db

EXPOSE 1433

COPY $PATH_TO_YOUR_BACKUP_FILE /var/opt/mssql/backup/$your_db_name.bak
RUN (/opt/mssql/bin/sqlservr --accept-eula & ) | grep -q "Server is listening on" && /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'YourPassword' -Q "RESTORE DATABASE $your_db_name FROM DISK='/var/opt/mssql/backup/$your_db_name.bak' WITH MOVE '$your_db_name' TO '/var/opt/mssql/data/$your_db_name.mdf', MOVE '$your_db_name_log' TO '/var/opt/mssql/data/$your_db_name_log.ldf'"

```

**Let's understand it line by line**

- `FROM mcr.microsoft.com/mssql/server:2017-latest` specifies base mssql image that we want to extend. I am using `2017-latest` version. you can use your choice of version. 

- **Environment Variables**
    - `ENV ACCEPT_EULA=y` to accept *End User License Agreement*
    - `ENV MSSQL_SA_PASSWORD=YourPassword` specifies your database system administrator password. replace `YourPassword` with your password.
    - `ENV MSSQL_PID=Express` specifies the mssql server edition or product key. *Default value = `Developer`*
  
- `EXPOSE 1433` specifies the port we want to expose to the docker for this image from within the container. Since mssql by default expose `1433`, so I left it as it is.

- `COPY PATH_TO_YOUR_BACKUP_FILE /var/opt/mssql/backup/your_db_name.bak` copies your back-up file on the specified location inside container. feel free to change the destination path.

- **Let's break `RUN` command**
    - 
