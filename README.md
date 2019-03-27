
###############################################################################
# DOWNLOAD SOFTWARE FROM OTN.ORACLE.COM (you need a login account)
# https://www.oracle.com/database/technologies/appdev/apex.html
###############################################################################

mkdir downloads

###############################################################################
# SET ENVIRONMENT
###############################################################################

DATABASE_VERSION=18.4.0
APEX_VERSION=18.2
ORDS_VERSION=18.4.0.354.1002
ORDS_PORT=8080

###############################################################################
# UNPACK SOFTWARE
###############################################################################

# create folders for the three components (ORACLE DATABASE, APEX, ORDS)
mkdir apex oradata ords
# oracle user inside container needs to have access to this folder
chmod 777 oradata

# unpack APEX in the background (it takes a while)
unzip ~/docker/oracle-apex-ords/downloads/apex_${APEX_VERSION}.zip -d ~/docker/oracle-apex-ords/apex/${APEX_VERSION} &

###############################################################################
# BUILD ORACLE DATABASE IMAGE
###############################################################################

# do we have our images ready?
docker image ls | grep -E 'oracle|ords'

# oracle database. Let's build on Gerald Venzl work...
git clone https://github.com/oracle/docker-images.git
# copy in binary
cp ./downloads/oracle-database-xe-18c-1.0-1.x86_64.rpm \
   ./docker-images/OracleDatabase/SingleInstance/dockerfiles/${DATABASE_VERSION}
cd ./docker-images/OracleDatabase/SingleInstance/dockerfiles/
# build the image. This is going to take a while... (10 minutes)
./buildDockerImage.sh -v ${DATABASE_VERSION} -x

###############################################################################
# BUILD APACHE TOMCAT IMAGE 
###############################################################################

# build the ords container with the recipe from Martin D Souza
cd ords
git clone https://github.com/martindsouza/docker-ords.git .
# copy in the ords.war file
unzip ../downloads/ords-18.3.0.270.1456.zip -d ./docker-ords/ords.war
cd docker-ords
# now build!
docker build -t ords:$ORDS_VERSION .


###############################################################################
# ORACLE DATABASE
###############################################################################

# create database. It will take a while...
docker-compose up database
# Ctrl+C to stop the database after "DATABASE IS READY TO USE"
# now start the database in detached mode
docker-compose up -d database
# make sure health is fine.
# if not, perform a restart: "docker-compose restart database"
docker-compose ps

# test login to the CDB from your host client
sqlplus system/oracle@localhost:1521/XE
# test login to the PDB from your host client
sqlplus system/oracle@localhost:1521/XEPDB1

###############################################################################
# INSTALL APEX IN XEPDB1
###############################################################################

cd ./apex/${APEX_VERSION}/apex
sqlplus "sys/oracle@localhost:1521/XEPDB1 as sysdba" @./../../../install_apex.sql
cd ./../../../

###############################################################################
# CONFIGURE ORDS
###############################################################################

# create the configuration folder for given ORDS version
mkdir -p ./ords/ords-${ORDS_VERSION}/config

# OnOff. Generate ORDS configuration
docker-compose up create-ords-config
# Ctrl+C
# start application (Tomcat configured with ORDS)
docker-compose up -d app

###############################################################################
# VIOLA! Point your browser to: http://localhost:8080/ords
# Workspace: INTERNAL, Username: ADMIN, Password: Welcome1
###############################################################################

open http://localhost:8080/ords

################################################################################
# CLEANUP
################################################################################

docker-compose down -v

rm -rf ./oradata/*
rm -rf ./ords/ords-${ORDS_VERSION}



h
