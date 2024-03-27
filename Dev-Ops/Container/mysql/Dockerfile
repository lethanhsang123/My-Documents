# Use the official MySQL image as the base image
FROM mysql:latest

# Enviroment variables for MySQL configuration 
ENV MYSQL_ROOT_PASSWORD=root
ENV MYSQL_DATABASE=product-service
ENV MYSQL_USER=admin
ENV MYSQL_PASSWORD=admin

# Copy custom configuration files or scripts (if need)
# COPY ./my.cnf /etc/mysql/my.cnf
# COPY ./init.sql /docker-entrypoint-initdb.d/init.sql

# Expose the MySQL port (it does't publish port to host machine)
EXPOSE 3306

# Optionally, set the working directory
# WORKDIR /app

# Command to run MySQL server
CMD ["mysqld"]



