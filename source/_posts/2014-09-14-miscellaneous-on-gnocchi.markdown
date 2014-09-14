---
layout: post
title: "Miscellaneous Resources on Gnocchi"
date: 2014-09-14 13:33:23 -0400
comments: false
categories: [Openstack, OPW]
---

Like the title says, this post is a collection of links to resources that I found helpful for
learning about Gnocchi.

+ Julien Danjou's [blog post](https://julien.danjou.info/blog/2014/openstack-ceilometer-the-gnocchi-experiment)
 about Gnocchi is the most recent of the links here and an excellent summary of all things Gnocchi. It explains the limitations of the previous API
in Ceilometer and describes how the current implementation (using Pandas/Swift) was chosen. 
I found the diagrams useful for understanding the relationship between entities and resources.
The post expands on the information given in the [wiki](https://wiki.openstack.org/wiki/Gnocchi), also a good reference.

+ Julien also did a [walkthrough](https://plus.google.com/events/ci928omo7od91c2q2c9iii58aig) of the Gnocchi source code:
very helpful for navigating the different parts of the project.

+ Gnocchi [specs](https://github.com/openstack/ceilometer-specs/blob/master/specs/juno/gnocchi.rst)
 (I referred to this primarily for examples of the HTTP request syntax). If you look through the history, you can see the shift
 in approach to retention policies - the timeseries length goes from being defined in terms of number of points
 to being defined in terms of a lifespan.

+ Eoghan Glynn's [thread](http://lists.openstack.org/pipermail/openstack-dev/2014-August/042080.html) on the Openstack
mailing lists clearly outlines the differences between Gnocchi and timeseries-oriented databases (like InfluxDB). 
It goes over the features of InfluxDB that make it a good backend option for Gnocchi though, such 
as the downsampling and retention policies.

+ The Gnocchi [repository](https://github.com/stackforge/gnocchi/tree/master/gnocchi).

<!-- more -->

Finally, I am also adding some old notes I made when first trying to understand the Gnocchi source code. I found it helpful to visually 
map out the filesystem structure (shown below) with comments on the different functional parts underneath.
The diagram is not up to date and shows only the sections I was most familiar with.
Although the comments below are rather simplistic and a bit embarrassing in retrospect, 
I thought I would post this in case a future OPW intern wants to know how other people
approached their projects.

	gnocchi/gnocchi/
	|--- carbonara.py
	|--- cli.py
	|--- indexer/
	|	|--- __init__.py
	|	|--- null.py
	|	|--- sqlalchemy.py
	|--- __init__.py
	|--- openstack/
	|--- rest/
	|	|--- app.py
	|	|--- __init__.py
	|--- service.py
	|--- storage/
	|	|--- __init__.py
	|	|--- null.py
	|	|--- swift.py
	|--- tests/
	|	|--- __init__.py
	|	|--- test_carbonara.py
	|	|--- test_indexer.py
	|	|--- test_rest.py
	|	|--- test_storage.py
	|--- gnocchi.egg-info
	|--- openstack-common.conf
	|--- README.rst
	|--- requirements.txt
	|--- run-mysql-tests.sh
	|--- run-postgresql-tests.sh
	|--- run-tests.sh
	|--- setup.cfg
	|--- setup.py
	
+ carbonara.py is the library that manipulates the timeseries. It is used in conjunction with the Swift storage driver.

+ rest/ - I spent a lot of time here, mostly in the EntityController get_measures section of the init file, which
 has the code for evaluating HTTP requests to the API. The app.py file uses pecan magic to create hooks
 for the configured storage drivers and indexers. It also sets the default port for the API.

+ storage/ - also looked at these files a lot when starting out. The driver-specific code is contained here. Eoghan Glynn added support for
InfluxDB in this [patch](https://review.openstack.org/#/c/103329/); Dina Belova added OpenTSDB [support](https://review.openstack.org/#/c/107986/).
As it was desirable to make the custom aggregation functions driver-independent, no changes were made to these files. However, it was useful
to look at the InfluxDB patch to get a sense of the way aggregation was implemented using that backend.

+ tests/ - The second most helpful thing in understanding the code (other than actually running the API and seeing query results, as well as logging intermediate steps in debug mode) was to look at the test cases.

+ README.rst - I think this was my first contribution. Eoghan suggested it as a first step and it was a great way for getting my feet wet.

+ setup.cfg - modify this file to add entry points for custom aggregation functions, storage drivers, and indexer options.

That's all I have for comments on the Gnocchi source code - thanks for reading!