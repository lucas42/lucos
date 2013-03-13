#lucos

lucos is a system of web-based modules, which do an array of different things.  I'm gradually putting each of these up on github bit-by-bit.

##Terminology
* **Service** A program, which when run acts as a webserver on a single port
* **Module** A unit of functionality provided by a service
Whilst most services provide a single module, some provide multiple (or none at all).

##Services
This is an incomplete list of lucos services:

* **[services](https://github.com/lucas42/lucos_services)** A program which is in charge of running all the other services.
* **[transport](https://github.com/lucas42/lucos_transport)** Keeps track of public transport data (in a manner suitable for forms of transport with poor networking conditions)
* **mediamanager** Keeps track of a playlist of music to play
* **mediaselector** Makes decisions about what music to play
* **root** Displays a list of lucos modules on a handy homepage
* **notes** Keeps track of a list of todo items
* **contacts** Keeps track of a database of people
* **auth** Authenticates users of lucos
* **time** Keeps track of the time
