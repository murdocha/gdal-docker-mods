# GDAL Dockerfile modifications

I've added the MSSQL Server Spatial driver to the linux alpine-normal and alpine-small Docker configuration files.
I _also_ added the npm/node install to the alpine-small docker configuration (you could skip this if you don't need it for a smaller image).

The original source files come from:
https://github.com/OSGeo/gdal/tree/master/gdal/docker

I also struggled mightily to keep the Git history for just this gdal/docker directory. I _wanted_ to "fork one directory from the GDAL repo", but couldn't figure it out initially. I _did_ eventually "succeed" in getting the gdal docker file history as a starting point for this repo. Not a true "fork/clone" of a directory, but I learned a lot! I'm not at _all_ sure this is the best approach, but it seems to work. See notes below.

# MS SQL Server modifications
- MS SQL Server vector driver doc page: 
https://gdal.org/drivers/vector/mssqlspatial.html#vector-mssqlspatial
- MS SQL Server driver linux/alpine installation docs:
https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server?view=sql-server-ver15#alpine17
- My notes on getting this working on the GDAL/OSGeo forum:
http://osgeo-org.1560.x6.nabble.com/gdal-dev-Alpine-Docker-build-with-ODBC-support-tt5456854.html
- This was also helpful:
https://stackoverflow.com/questions/51888064/install-odbc-driver-in-alpine-linux-docker-container

I found that I had to not only add the "unixodbc-dev" library during GDAL build stage, but _also_ add the "unixodbc" _AND_ explicit MS SQL ODBC drivers to the final alpine build stage in the Dockerfile. Not absolutely sure that there is not a way to avoid using the unixodbc drivers, but this method worked!

- Added a modified "alpine-small-mssql-node" Docker image based on:
https://github.com/OSGeo/gdal/tree/master/gdal/docker#small-osgeogdalalpine-small-latest
- Added a modified "alpine-normal-mssql" Docker image based on:
https://github.com/OSGeo/gdal/tree/master/gdal/docker#normal-osgeogdalalpine-normal-latest


# Docker reference commands
build "alpine-normal" custom docker container (with MS SQL added, by default with Python and PostgreSQL support). Building Docker can take a while...
```
docker build -t local-gdal-build:alpine-normal-mssql . >dockerbuild-alpine-normal-mssql.log
```

build "alpine-small" custom docker container (with MS SQL and NPM/Node.js added, by default with PostgreSQL support)
```
docker build -t local-gdal-build:alpine-small-mssql-node . >dockerbuild-alpine-small-mssql-node.log
```

run interactive shell on Docker container with mounted directory "volume"
```
docker run --rm -it -v c:\projects\directory_to_mount:/mounted_directory local-gdal-build:alpine-normal-mssql /bin/sh
```

# GDAL reference commands (once you are attached to the Docker container via shell/terminal)
list available vector OGR formats within Docker GDAL container using "ogrinfo" command:
https://gdal.org/programs/ogrinfo.html?highlight=ogrinfo
You _should_ now see MS SQL Spatial and ODBC formats show up here

```
ogrinfo --formats
```

query "SomeTableName" MS SQL Server table (you _could_ put this all in a local shell script that you reference from the mounted volume).
```
export MSHOST=<some_db_server_location>
export MSPORT=1433
export MSDatabase=<some_db_name>
export MSSchema=<some_db_schema>
export MSDriverName="ODBC Driver 17 for SQL Server"
export MSUSER=<some_db_username>
export MSPASSWORD="<some_db_password>"
export sqlconnectionstring="MSSQL:SERVER=${MSHOST},${MSPORT};DATABASE=${MSDatabase};UID=${MSUSER};PWD=${MSPASSWORD};DRIVER=""${MSDriverName}"

ogrinfo -al "${sqlconnectionstring};TABLES=${MSSchema}.SomeTableName"
```

run local sh script from within docker container
```
/bin/sh /mounted_directory/test.sh
```

# Issues getting a Git clone of just a single directory from a larger repo

I am only an intermediate Git user, and so I was unable to easily clone _just_ the desired "docker" directory from the main GitHub GDAL repository (https://github.com/OSGeo/gdal/tree/master/gdal/docker).
I originally just copied the relevant bits here (as of 19 Feb 2021) and modified them in subsequent commits. But I didn't like that approach... I tried a _lot_ of methods to clone just a single directory's contents to keep the git history for just this directory of interest, and _EVENTUALLY_ got something working! (maybe?)

### Initial attempt (failure!):
I _thought_ this might provide a solution (with "modern" Git features, after updating my Git for Windows install to latest 2.30.1.windows.1):
https://askubuntu.com/a/1074185
```
git clone --depth 1 --filter=blob:none --no-checkout https://github.com/OSGeo/gdal
cd gdal
git checkout master -- gdal/docker
```

But there were a _lot_ of "orphaned" blob object references that prevented me from actually pushing the results up to my new personal GitHub repo, which prevented a successful push (fatal errors) when attempting to push to my new remote:
```
git remote rm origin
git remote add origin https://github.com/<your-github-repo-here>.git
git remote -v
git push -u origin master
```

### Subsequent attempt (success!):
This post did the trick:
https://github.community/t/adding-a-folder-from-one-repo-to-another/781/2
which refers to an older post here:
http://gbayer.com/development/moving-files-from-one-git-repository-to-another-preserving-history/

I was able to create a "subset repo" that only showed the directory of interest.
It (initially) also retained 240MB of Git history when viewing the size of the ".git" folder... but the secondary steps trim down the Git history to just the desired directory's changes!


These were the commands I used successfully:
```
cd c:\projects\gdal-docker-stuff
mkdir backup
cd backup
git clone https://github.com/OSGeo/gdal.git
cd gdal
git remote rm origin
git filter-branch --subdirectory-filter gdal/docker -- --all
```
At that point, the Git history still has ~240MB of history in the ".git" folder/blackbox... This gets stripped down in subsequent steps!

I didn't care to make the directory structure match with a nested "gdal/docker" directory, so I skipped the moving directory steps in the link posted above.

Then these steps to create a new repo as a destination:

```
cd c:\projects\gdal-docker-stuff
mkdir destination
cd destination
git init
```

Then these steps to transfer selected directory and history to the new destination repo:

```
git remote add modified-source c:\projects\gdal-docker-stuff\backup\gdal\docker
git pull modified-source master --allow-unrelated-histories
```

Down to a Git history (.git file size) of just 138kb! Some progress!
Then I moved the entire destination folder (with .git folder) to my desired folder name and location.
and added the new remote to my GitHub page (this online repo).

```
git remote add origin https://github.com/murdocha/gdal-docker-mods2.git
git push -u origin main
```

Boom! new baseline for my repo to just show the content and history from the gdal/docker directory! (but sadly, not a true "fork". But that's ok for my purpose here.)

Make my new edits (via copy/paste). Make separate commits. Push changes and done.
