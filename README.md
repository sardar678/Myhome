Legrand (BTicino) MyHome plugin for homebridge: https://github.com/nfarina/homebridge

Legrand MyHome (http://www.homesystems-legrandgroup.com/BtHomeSystems/home.action) is an Home Automation solution that can manage:

lighting (standard on/off/dimmed lights)
thermoregulation
curtains, doors
security systems
With this plugin, the support of a IP gateway installed in your plant and a configuration of all installed systems (MyHome does not support the autodiscovery of the system) you can control it. You need to disable the OpenWebNet password-based authentication from the IP of the device that runs homebridge (ie. Raspberry) or set the auhentication to HMAC; HMAC authentication is supported by all recent IP gateways or older one with updated firmware (eg. F454 with v2 firmware).
