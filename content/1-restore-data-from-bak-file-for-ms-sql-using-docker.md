Title: Restore data from bak-file for MS-SQL using Docker
Date: 2022-12-05 10:20
Category: How-To?
Tags: docker, mssql, how-to

> Do you want to restore data from a backup file and don't want to use full blown mssql server in local?  
> **How about creating docker image for it?** 

## Why I am ditching SSMS/mssql server studio

1. Full blown installation 
2. For setting up fresh DB, it needs a complete clean up.
3. I just wanted to restore data and want to use it in my python app.

## Write Dockerfile

Open a text editor and copy below content into `Dockerfile` 

```Dockerfile
FROM mcr.microsoft.com/mssql/server:2017-latest

ARG absolute_path_to_your_backup_file
ARG your_db_name
ARG your_password

ENV ACCEPT_EULA=y
ENV MSSQL_SA_PASSWORD=$your_password
ENV MSSQL_PID=Express

EXPOSE 1433

COPY $absolute_path_to_your_backup_file /var/opt/mssql/backup/$your_db_name.bak
RUN (/opt/mssql/bin/sqlservr --accept-eula & ) | \
    grep -q "Server is listening on" \
    && /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P '$your_password' \
        -Q "RESTORE DATABASE $your_db_name FROM DISK='/var/opt/mssql/backup/$your_db_name.bak' \n\
        WITH MOVE '$your_db_name' TO '/var/opt/mssql/data/$your_db_name.mdf', \n\
        MOVE '$your_db_name_log' TO '/var/opt/mssql/data/$your_db_name_log.ldf'"

```

**Let's understand it line by line**

- `FROM mcr.microsoft.com/mssql/server:2017-latest` specifies base mssql image that we want to extend. I am using `2017-latest` version. you can use the version of your choice. 
  
- **Arguments** These arguments you have to pass during docker build.
    - `ARG absolute_path_to_your_backup_file` is the argument for the absolute path of your backup file.
    -  `ARG your_db_name` is the argument for your db name.
    -  `ARG your_password` is the argument for your password.

- **Environment Variables**
    - `ENV ACCEPT_EULA=y` to accept *End User License Agreement*
    - `ENV MSSQL_SA_PASSWORD=$your_password` specifies your database system administrator password, which will be passed during docker build
    - `ENV MSSQL_PID=Express` specifies the mssql server edition or product key. *Default value = `Developer`*
  
- `EXPOSE 1433` specifies the port we want to expose from within the container for this image. Since mssql by default expose `1433`, so I left it as-is.

- `COPY $absolute_path_to_your_backup_file /var/opt/mssql/backup/$your_db_name.bak` copies your back-up file on the specified location inside container. 

- **Let's break `RUN` command**
    - `/opt/mssql/bin/sqlservr --accept-eula &` will run `sqlservr` script with the required `--accept-eula` agreement which will start the server.
    > Q. why do we need to run the server?  
    > A. because we want can `RESTORE DATABASE` sql query.
    - `&` specifies this script will run in the background.
    - `| grep` will wait and search the logs for `"Server is listening on"`, to occur.  
        Once it appears `/opt/mssql-tools/bin/sqlcmd` will run `sqlcmd` script which will take args:
        - `-S localhost` for host
        - `-U sa` for username
        - `-P '$your_password'` for password
        - `-Q "RESTORE DATABASE $your_db_name FROM DISK='/var/opt/mssql/backup/$your_db_name.bak' WITH MOVE '$your_db_name' TO '/var/opt/mssql/data/$your_db_name.mdf', MOVE '$your_db_name_log' TO '/var/opt/mssql/data/$your_db_name_log.ldf'"`  
        which will execute SQL query to restore data from the database backup file.
  
## Build Docker Image

Now since we have created Dockerfile, Let's build the image using below command:
```sh
docker build \ 
    --build-arg \ 
    absolute_path_to_your_backup_file=<your bak file path> \ 
    your_db_name=<your db name> \ 
    your_password=<your password> \ 
    -t <name>:latest .
```
- `--build-arg` specifies the argument you need to pass during build.
- `<name>` specify the name of your image. 
- `latest` specifies the tag, you can specify the version of you image. It is optional.
- `.` is the directory containing the `Dockerfile` we created