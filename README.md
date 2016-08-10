# gnaf

## Introduction
This project provides:

1. gnaf-createdb: scripts to load the [G-NAF data set](http://www.data.gov.au/dataset/geocoded-national-address-file-g-naf) into a relational database.
2. gnaf-common: database and utility code shared by 3 & 4.
3. gnaf-indexer: a [Scala](http://scala-lang.org/) program to query the database to produce JSON and scripts to load this into [Elasticsearch](https://www.elastic.co/).
4. gnaf-test: a [Scala](http://scala-lang.org/) program to query the database to produce JSON address data
and a [Node.js](https://nodejs.org/en/) script to query Elasticsearch for these addresses and report on where the correct result is found in the results
(we expect it to be the first hit most of the time, even when there are deliberate small errors in the input address). 
4. gnaf-service: a [Scala](http://scala-lang.org/) [RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) web service providing
access to the G-NAF database.
5. gnaf-contrib: a [Scala](http://scala-lang.org/) [RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) web service providing
access to the gnafContrib database of user supplied geocodes.
6. gnaf-ui: static files providing a demonstration web user interface using Elasticsearch, gnaf-service and gnaf-contrib.

The top level directory contains no code, but contains the [sbt](http://www.scala-sbt.org/) build for 2-5 (no build is required for 1 & 6).

## Install Tools

To run the Scala code install:
- a JRE e.g. from openjdk-8 (version 8 or higher is required by some dependencies);
- the build tool [sbt](http://www.scala-sbt.org/).

To develop [Scala](http://scala-lang.org/) code install:
- the above items (you may prefer to install the full JDK instead of just the JRE but I think the JRE is sufficient);
- the [Scala IDE](http://scala-ide.org/download/current.html).

## Build

Automatic builds are available at: https://t3as-jenkins.it.csiro.au/job/gnaf-master/ (only within the CSIRO network).

The command:

    sbt clean one-jar dumpLicenseReport

from the project's top level directory cleans out previous build products,
builds one-jar files (which include all dependencies) for stand-alone executables and 
creates license reports on dependencies.

## Run

This section provides a very brief summary of how to run the project. Detailed information is available in the sub-project README.md files.

	cd gnaf-createdb
	# create SQL load script
	script/createGnafDb.sh
	# run h2 with postgres protocol
	java -Xmx3G -jar ~/.ivy2/cache/com.h2database/h2/jars/h2-1.4.191.jar -web -pg &
	# create and load database (takes about 90 minutes with a SSD)
	psql --host=localhost --port=5435 --username=gnaf --dbname=~/gnaf < data/createGnafDb.sql
	Password for user gnaf: gnaf
	# kill h2
	kill %1
	cd ..
	
	# start Elasticsearch - see gnaf-indexer/README.md and Elasticsearch documentation
	# run indexer (which uses the database in embedded mode so will fail if the above h2 process is still running; requires jq; takes about 2 hours)
	cd gnaf-indexer
	src/main/script/loadElasticsearch.sh
	cd ..
	
	# test Elasticsearch index
	# get index schema
	curl 'http://localhost:9200/gnaf/?pretty'
	# search for an address (no punctuation except for '-' as number range separator, see gnaf-indexer/README.md for details)
	curl -XPOST 'localhost:9200/gnaf/_search?pretty' -d '
    {
        "query": {
            "match": {
                "d61Address": {
                    "query": "7 LONDON CIRCUIT CITY ACT 2601",
                    "fuzziness": 2,
                    "prefix_length": 2
                }
            }
        },
        "size": 5
    }'
    
	# start gnaf database web service (uses gnaf database in embedded mode)
	nohup java -jar gnaf-service/target/scala-2.11/gnaf-service_2.11-0.1-SNAPSHOT-one-jar.jar &> gnaf-service.log &
	
	# test gnaf-service
	# use this in swagger-ui
	curl 'http://localhost:9000/api-docs/swagger.json
	# get geocode types and descriptions
	curl 'http://localhost:9000/gnaf/geocodeType'
	# get type of address e.g. RURAL, often missing, for an addressDetailPid
	curl 'http://localhost:9000/gnaf/addressType/GANSW716635201'
	# get all geocodes for an addressDetailPid, almost always 1, sometimes 2
	curl 'http://localhost:9000/gnaf/addressGeocode/GASA_414912543'
	
	# start contrib database web service (uses gnafContrib database in embedded mode)
	nohup java -jar gnaf-contrib/target/scala-2.11/gnaf-contrib_2.11-0.1-SNAPSHOT-one-jar.jar &> gnaf-contrib.log &
	
	# test gnaf-contrib
	# use this in swagger-ui
	curl 'http://localhost:9010/api-docs/swagger.json
	#  add contributed geocode for an addressSite
	curl -XPOST 'http://gnaf.it.csiro.au:9010/contrib/' -H 'Content-Type:application/json' -d '{
	  "contribStatus":"Submitted","addressSitePid":"712279621","geocodeTypeCode":"EM",
	  "longitude":149.1213974,"latitude":-35.280994199999995,"dateCreated":0,"version":0
	}'
	# list contributed geocodes for an addressSite
	curl 'http://gnaf.it.csiro.au:9010/contrib/712279621'
	# there are also delete and update methods
	
Test with:
- demonstration web user interface by opening the file `gnaf-ui/html/index.html` in a recent version of Chrome, Firefox or Edge;
- [swagger-ui](http://swagger.io/swagger-ui/) available at http://gnaf.it.csiro.au/swagger-ui/ (only within the CSIRO network) using one of the above swagger.json URLs.
	
## Nginx Configuration

The setup described here allows the webapp to be tested on a mobile device (with GPS) by avoiding the csiro/corporate network.
Firefox on android currently allows access to location by webapps served with HTTP (Chrome only permits this with HTTPS).

Create a file `/etc/nginx/sites-available/gnaf`:

	server {
	    listen 80 default_server;
	    listen [::]:80 default_server;
	    root /var/www/html;
	    index index.html;
	    server_name _;

	    # serve elasticsearch with CORS header on port 80 path /es 
	    location /es {
			proxy_pass http://127.0.0.1:9200/gnaf;
			add_header Access-Control-Allow-Headers 'Content-Type';
	    }

	    location / {
			# First attempt to serve request as file, then
			# as directory, then fall back to displaying a 404.
			try_files $uri $uri/ =404;
	    }
	}
	
Enable the gnaf config and disable the default config:

	cd /etc/nginx/sites-enabled/
	sudo ln -s ../sites-available/gnaf .
	sudo rm default
	
Copy content of `gnaf-ui/html` to `/var/www/html`.
Edit `initBaseUrl()` in the copied `index.js` if necessary to use the above nginx proxy for Elasticsearch. 
	
Reload with `sudo /etc/init.d/nginx reload` (or restart).

Now access the webapp from the mobile device:

- start the portable hotspot on the mobile device
- connect to it from the server noting the server's IP address on the hotspot e.g. 192.168.43.207
- in Firefox on the mobile device open the root page at the server's IP address e.g.: http://192.168.43.207/

## Develop With Eclipse

The command:

    sbt update-classifiers eclipse

uses the [sbteclipse](https://github.com/typesafehub/sbteclipse/wiki/Using-sbteclipse) plugin to create the .project and .classpath files required by Eclipse (with source attachments for dependencies).

## Software License

This software is released under the CSIRO BSD license - see `Licence.txt`.
Each of the sub-projects 2 - 4 lists its dependencies and their licenses in `3rd-party-licenses.html`.

## Data License

Incorporates or developed using G-NAF ©PSMA Australia Limited licensed by the Commonwealth of Australia under the
[Open Geo-coded National Address File (G-NAF) End User Licence Agreement](http://data.gov.au/dataset/19432f89-dc3a-4ef3-b943-5326ef1dbecc/resource/09f74802-08b1-4214-a6ea-3591b2753d30/download/20160226---EULA---Open-G-NAF.pdf).

