# lucos

lucos is a system of web-based modules, which do an array of different things.  I'm gradually putting each of these up on github bit-by-bit.

## Terminology
* **Service** A program, which when run acts as a webserver on a single port
* **Module** A unit of functionality provided by a service
Whilst most services provide a single module, some provide multiple (or none at all).

## Services
This is an incomplete list of lucos services:

* **[services](https://github.com/lucas42/lucos_services)** A program which is in charge of running all the other services.
* **[transport](https://github.com/lucas42/lucos_transport)** Keeps track of public transport data (in a manner suitable for forms of transport with poor networking conditions).
* **[dns](https://github.com/lucas42/lucos_dns)** A lightweight dynamic DNS tool.
* **[notes](https://github.com/lucas42/lucos_notes)** An offline todo list.
* **[mediamanager](https://github.com/lucas42/lucos_media_manager)** Keeps track of a playlist of music to play
* **[mediaselector](https://github.com/lucas42/lucos_media_selector)** Makes decisions about what music to play
* **[root](https://github.com/lucas42/lucos_root)** Displays a list of lucos modules on a handy homepage
* **[contacts](https://github.com/lucas42/lucos_contacts)** Keeps track of a database of people
* **[auth](https://github.com/lucas42/lucos_authentication)** Authenticates users of lucos
* **[time](https://github.com/lucas42/lucos_time)** Keeps track of the time

## Libraries
* **[navbar](https://github.com/lucas42/lucos_navbar)** Navigation bar, giving consistent styling across apps.
* **[time_component](https://github.com/lucas42/lucos_time_component)** Displays the time, NTP-synced to lucos_time
