# Lucos v4 architecture

* Containerise services
* Discovery mechanism
* Surface info, following discovery - json endpoint?
	- Routing
		- DNS
		- Port running on
	- Stuff for displaying homepage:
		- Service name
		- URL
		- Icon
		- Whether service works offline
	- Monitoring?


## Migration strategy

* Create new VM on hive for housing containers
* Start with things which encounter operational problems (eg tfluke app)
* Next focus on legacy stuff on prometheus (eg DNS)
* Then anything not on daedalus
* Finish with remaining daedalus services
* Once completed, could look at removing virtualisation layer and running containers on bare metal