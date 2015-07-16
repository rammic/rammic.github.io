---
layout: post
title: "Configuring CouchDB-Lucene with Tomcat"
description: "Configuring CouchDB-Lucene with Tomcat"
category: development
tags: [couchdb,lucene,couchdb-lucene]
---
{% include JB/setup %}

I had fun trying to get CouchDB-Lucene working with Tomcat Server (7.X). I wanted it running in Tomcat since I’m quite familiar Tomcat’s care and feeding (i.e. install/configure/upgrade routines). It also helps that I’ve always had terrible luck getting any CLI utility, like the one provided by CouchDB-Lucene, running reliably as a service. I finally got it working- Here are my notes from that endeavor.


#Background
-----

The way CouchDB-Lucene (I’ll call it CDBLucene here for brevity) works is that you configure your CouchDB instance to proxy certain HTTP requests, specifically requests for the _fti path, to the CDBLucene instance. CDBLucene is then responsible for returning the search results.

CDBLucene, of course, can’t do this alone. It has to be configured to know where the parent CouchDB instance is located and how to connect to it.

![CouchDB-Lucene-Tomcat](/img/cdblucene-tomcat.png)
*My Crude Illustration of the CouchDB-Lucene Request/Response Flow*

Note that it’s entirely possible (and probably desirable in many situations) to query CDBLucene directly for the results and bypass the additional load and overhead on the CouchDB instance.

![CouchDB-Lucene-Tomcat](/img/cdblucene-tomcat-direct.png)
*Alternate (Direct) Use of CDBLucene- Bypasses CouchDB HTTP Proxying*

I wanted to have our primary CouchDB 1.6.1 instance run on a separate VM than the CDBLucene instance. Since they run on separate JVMs and communicate using HTTP, this should be technically possible.

To do so, we need to:

* **Configure CouchDB → CDBLucene**: Configure CouchDB to forward requests to CDBLucene, and
* **Configure CDBLucene → CouchDB**: Configure CDBLucene to know how to reach back to CouchDB to read the appropriate index

Again, note that step 2 is entirely optional. You can simply query CDBLucene directly since CouchDB is nothing more than a glorified HTTP proxy for lucene searches.

Assumptions We had two Ubuntu VMs already set up and running in AWS. One had our existing installation of CouchDB, complete with an existing database and design view with a lucene index already created. The second VM that will host the CDBLucene instance was freshly created and running Tomcat as an upstart service (installed courtesy of apt-get with the tomcat7 image).

#Configuring CDBLucene → CouchDB 
This is what gave me a minor headache. The CDBLucene documentation is non-existent, so I had to jump through the source a bit to figure out what’s going on. Here’s what I ended up doing:

From the machine you intend to host the CDBLucene instance, clone the GitHub repository.

`git clone https://github.com/rnewson/couchdb-lucene.git`

This will create a clone in a new couchdb-lucene directory. Once completed, we need to make some modifications since CDBLucene depends on an embedded couchdb-lucene.ini that gets rolled up into the war file. Edit the couchdb-lucene/src/main/resources/couchdb-lucene.ini file to reflect your configuration, making sure the url parameter points back to your CouchDB instance.

{% highlight ini %}
[lucene]
port=8080
host=[cdb_lucene-instance]
dir=lucene-indexes
timeout=10000
limit=25
allowLeadingWildcard=false

[couchdb-lucene]
url=http://[couchdb-instance]:5984
{% endhighlight %}

The `[lucene]` section refers to the CDBLucene instance, so include the host and port that you’ve configured Tomcat to use. Also note that the dir path will refers to `[TOMCAT_HOME]` as the base path, so you may want to change this or create `[TOMCAT_HOME]/lucene-indexes` directory manually.

You now should build the package. Change into the couchdb-lucene directory and start a maven build including the war:war goal:

{% highlight bash %}
cd couchdb-lucene
mvn clean install war:war
{% endhighlight %}

This will build a deployable war file in the /target directory. Since by default will create a war file with the full version name, I renamed it so that it’s a bit more memorable when you go to deploy it:

{% highlight bash %}
mv target/couchdb-lucene-1.1.0-SNAPSHOT.war couchdb-lucene.war
{% endhighlight %}

Now you can deploy the war to file to Tomcat by moving it to the [TOMCAT_HOME]/webapps directory. Assuming that you installed Tomcat using apt-get, this is as simple as:

{% highlight bash %}
cd couchdb-lucene.war /var/lib/tomcat7/webapps/
{% endhighlight %}

Tomcat should immediate detect the new file and move to hot-deploy it. Check the Tomcat instance to make sure the deployment occurred:

{% highlight bash %}
curl -L localhost:8080/couchdb-lucene
{% endhighlight %}

You should be able to connect but will receive a ‘bad_request’ reason:

{% highlight json %}
{
	"reason":"bad_request",
	"code":400
}
{% endhighlight %}


The bad_request reason is expected since we didn’t provide any useful parameters. You should also check the couchdb-lucene logs to make sure the indexing was started correctly:

{% highlight bash %}
cat /var/log/tomcat7/couchdb-lucene.log
{% endhighlight %}

If successful, you should see an entry like:

{% highlight bash %}
Index output goes to: /var/lib/tomcat7/lucene-indexes
{% endhighlight %}

Great! Now back over to the CouchDB instance to finish the installation:

#Configuring CouchDB → CDBLucene 

This is actually pretty straight forward. As described in the CouchDB-Lucene docs, you’ll want to go into your CouchDB local.ini file (/etc/couchdb/local.ini on Ubuntu) and add a httpd_global_handlers reference to point to your CDBLucene Tomcat instance.

{% highlight ini %}
[httpd_global_handlers]

_fti = {couch_httpd_proxy, handle_proxy_req, <<”http://[TOMCAT_INSTANCE]/couchdb-lucene">>}
{% endhighlight %}

Once done, restart CouchDB and try to run a search:

{% highlight bash %}
curl -l http://[couchdb]:8080/_fti/[database]/_design/[view]/[search]?q=[search_query]
{% endhighlight %}

Note that the request may timeout the first time because CDBLucene is building its index.

#Summary 


**TL;DR:** The primary trick to this is to realize that most of the settings you need to separate CDBLucene and CouchDB reside in the CDBLucene packaged war. The easiest way to make necessary configuration changes is to edit the `/src/main/resources/couchdb-lucene.ini` file prior to packaging with maven. 

If you forget or need to change it later, you can do so post-deployment by going into the `[TOMCAT_HOME]/webapps/couchdb-lucene/WEB-INF/classes` folder and editing `couchdb-lucene.ini` there. Restart tomcat and you should be ready to go (just be careful since any redeployment will overwrite this file).