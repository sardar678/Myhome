# Myhome
Legrand Myhome automation 
const debug = require('debug')('myhomeclient')
const sprintf = require("sprintf-js").sprintf;
const net = require('net');
const crypto = require('crypto');
const sha256 = require('sha256'), sha1 = require('sha1');



function inArray(needle, haystack) {
    var length = haystack.length;
    for(var i = 0; i < length; i++) {
        if(haystack[i] == needle)
            return true;
    }
    return false;
}
/*
 * You can test and debug with 
 * DEBUG=myhomeclient node script.js
 */

class MyHomeClient {

    constructor(_ipaddress, _port, _password, _classToCallback) {
        // Ten level translation, 0=0=Off, 1 = 100=On, 2=1%, 3=10%, 4=20%...8=60%,9=75,10=100%
        this.dimmerLevels = [0,100,1,10,20,30,40,50,60,75,100];
        this.buffers = {};
        this.lightBuses = [];
        this.ipaddress = _ipaddress;
        this.password = _password;
        this.port = _port;
        this.parent = _classToCallback;
    }

    start() {
        this.monitor = new MyHomeConnection(this.ipaddress, this.port, this.password, "MONITOR", null, this.onMonitor.bind(this));
        this.command = new MyHomeConnection(this.ipaddress, this.port, this.password, "COMMAND", this.onCommandConnect.bind(this), this.onCommand.bind(this));
    }

    /**
     * Fill the this.lightBuses array to know how many bus(es) should be polled at the startup
     */
    addLightBusDevice(_address) {
        var address = _address.split("/"); 
        if (address.length != 3) return "";

        // Add this bus if it wasn't already in list
        if (!inArray(address[0],this.lightBuses)) this.lightBuses.push(address[0]);
    }

    send(_command) {
        this.command.send(_command);
    }

    _slashesToAddress(_address) {
        var address = _address.split("/"); 
        if (address.length != 3) return "";

        var b = parseInt(address[0]), a = parseInt(address[1]), pl = parseInt(address[2]);

        if (b == 0) {
            if (a >= 10 || pl >= 10) {
                return sprintf("%02d%02d",a,pl);
            }
            return sprintf("%d%d",a,pl);
        } else {
            if (a >= 10 || pl >= 10) {
                return sprintf("%02d%02d#4#%02d",a,pl,b);
            }
            return sprintf("%d%d#4#%02d",a,pl,b);
        }
    }

    _addressToSlashes(_address) {
        if (_address.length == 4) return sprintf("0/%d/%d", parseInt(_address.substring(0,2)),parseInt(_address.substring(2,4)))
        if (_address.length == 2) return sprintf("0/%d/%d", parseInt(_address.substring(0,1)),parseInt(_address.substring(1,2)))
        // TODO
        return "9/99/99";
    }

    /*
     * Avoid repeated calls of the same command on a short time, it can happen with the dimmer control on iOS 
     * It buffer the commands and sends the last one.
     */
    _bufferCommand(_class,_address,_command, _timeout) {
        if (this.buffers[sprintf("%s-%s",_class,_address)] != null) {
            clearTimeout(this.buffers[sprintf("%s-%s",_class,_address)]);
        }
        this.buffers[sprintf("%s-%s",_class,_address)] = setTimeout(function() {
            this.command.send(_command);
            delete(this.buffers[sprintf("%s-%s",_class,_address)]);
        }.bind(this), _timeout);
    }

    relayCommand(_address,_on) {
        var address = this._slashesToAddress(_address);
        if (address == "") return;
        this.command.send(sprintf("*1*%d*%s##",(_on)?1:0,address));
    }

    simpleBlindCommand(_address,_stopUpDown) {
        var address = this._slashesToAddress(_address);
        if (address == "") return;

        /* Send a starting stop */
        if (_stopUpDown != 0) {
            this.command.send(sprintf("*2*%d*%s##",0,address));
        }
        this.command.send(sprintf("*2*%d*%s##",parseInt(_stopUpDown),address));
    }   

    advancedBlindCommand(_address,_shutterLevel) {
        var address = this._slashesToAddress(_address);
        if (address == "") return;

        this.command.send(sprintf("*#2*%s*#11#001*%d##",address,parseInt(_shutterLevel)));
    }       

    dimmerCommand(_address,_bri) {
		// 2013.04.12 - angeloxx - LG explains speed parameter of the *#1*<where>*#1*<level>*<speed> OWN frame
		// Speed is the single-step-time in 10ms scale. So the ramp time is:
		// time = <speed> * 10ms * <delta>, where delta is abs(currentlevel-finallevel).
	    // speed = (<time(ms)> / 10ms) / <delta>, delta level is [current|final]level*1000, ramptime is in ms
        
        var address = this._slashesToAddress(_address);
        if (address == "") return;

        if (_bri > 0) {
            this._bufferCommand("DIMMER",address,sprintf("*#1*%s*#1*%s*1##",address,_bri+100),500);
        } else {
            this._bufferCommand("DIMMER",address,sprintf("*1*0*%s##",address),500);
        }
    }

    // scenario activation (dendeps code)
    sceneCommand(_address,_scene) {
        var address = this._slashesToAddress(_address);
        if (address == "") return;
        this.command.send(sprintf("*0*%d*%s##",parseInt(_scene),address));
    }

    setSetPoint(_address,_temperature) {
        // Standard thermostat *4*40*%02d##*#4*#%02d*#14*%04d*3
        this._bufferCommand("THERMO",_address,sprintf("*4*40*%02d##*#4*#%02d*#14*%04d*3##*#4*%02d*14##",_address,_address,_temperature * 10,_address),500);

        // 4 Zones *#4*#0#%02d*#14*%04d*3
    }

    /**
     * Notify the availability of command connection to the client and
     * sends a light status request for all known buses
     */
    onCommandConnect() {
        this.lightBuses.forEach(function(element) {
            if (element == 0) {
                this.command.send("*#1*0##");
            } else {
                this.command.send(sprintf("*#1*0#4#%02d##",element));
            }
        }, this);

        /* Light status well keep some seconds, so I prefer to wait to send the connect feedback */
        setTimeout(function() {
            if (this.parent != null && this.parent.onConnect != null && typeof(this.parent.onConnect) == 'function') this.parent.onConnect();
        }.bind(this),this.lightBuses.length * 2500);
    }

    onMonitor(_frame) {
        // Split frame (TODO: and join it if is a long one that is splitted in multiple input)
        var frames = _frame.split("##");
        frames.forEach(function(frame) {
            if (frame == "") return;
            if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onMonitor) == 'function') this.parent.onMonitor(frame);

            /* Try to decode as... */
            var extract;

            /* Light level */
            extract = frame.match(/^\*1\*(\d+)\*([0-9#]+)$/);
            if (extract) {
                var address = this._addressToSlashes(extract[2]);
                if (extract[1] <= 1) {
                    if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onRelay) == 'function') this.parent.onRelay(address,extract[1] == 1);
                    debug(sprintf("LIGHTLEVEL %s ADDRESS %s",extract[1],address));
                } else {
                    if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onDimmer) == 'function') this.parent.onDimmer(address,this.dimmerLevels[parseInt(extract[1],10)]);
                }
            }

            /* Light level - Advanced (VantageControls) way */
            // *#1*12*1*190*1##>
            extract = frame.match(/^\*#1\*([0-9#]+)\*\d\*(\d+)\*\d+$/);
            if (extract) {
                var address = this._addressToSlashes(extract[1]);
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onDimmer) == 'function') this.parent.onDimmer(address,parseInt(extract[2],10)-100);
            }


            /* Simple (F411) Blind */
            extract = frame.match(/^\*2\*(\d)\*([0-9#]+)$/);                 
            if (extract) {
                var address = this._addressToSlashes(extract[2]);
                debug(sprintf("BLIND %s ADDRESS %s",extract[1],address));
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onSimpleBlind) == 'function') this.parent.onSimpleBlind(address,parseInt(extract[1],10));
            }

            /* Advanced (F401) Blind (on stop) */
            extract = frame.match(/^\*#2\*([0-9#]+)\*10\*(\d+)\*(\d+)\*\d+\*\d+$/);                 
            if (extract) {
                var address = this._addressToSlashes(extract[1]);
                var position = parseInt(extract[3],10);
                var direction = "STOP";

                if (parseInt(extract[2],10) == 11) direction = "UP";
                if (parseInt(extract[2],10) == 12) direction = "DOWN";

                debug(sprintf("ADVANCEDBLIND %s/%s ADDRESS %s",position,direction,address));
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onSimpleBlind) == 'function') this.parent.onAdvancedBlind(address,direction,position);
            }

            /* Ambient temperature */
            extract = frame.match(/^\*#4\*(\d+)\*[0|14]\*(\d\d\d\d)$/);
            if (extract) {
                var address = parseInt(extract[1]);
                var temperature = parseFloat(extract[2]/10);
                if (temperature > 100) {
                    temperature = -(temperature - 100);
                }
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onThermostat) == 'function') this.parent.onThermostat(address,"AMBIENT",temperature);
                debug(sprintf("AMBIENT %s ADDRESS %s",temperature,address));
            }

            /* Setpoint temperature */
            extract = frame.match(/^\*#4\*(\d+)\*12\*(\d\d\d\d)\*(\d+)$/);
            if (extract) {
                var address = parseInt(extract[1]);
                var temperature = parseFloat(extract[2]/10);
                if (temperature > 100) {
                    temperature = -(temperature - 100);
                }
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onThermostat) == 'function') this.parent.onThermostat(address,"SETPOINT",temperature);
                debug(sprintf("SETPOINT %s ADDRESS %s",temperature,address));
            }

            // External sensor (see http://www.myopen-legrandgroup.com/community/italian_my_open/italian_65536/f/152/p/1476/7590.aspx#7590)
            // Format *#4*<address>00*15*1*<temp-%04d>*0001##
            extract = frame.match(/^\*#4\*(\d+)00\*15\*1\*(\d\d\d\d)\*0001$/);
            if (extract) {
                var address = parseInt(extract[1]);
                var temperature = parseFloat(extract[2]/10);
                if (temperature > 100) {
                    temperature = -(temperature - 100);
                }
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onThermometer) == 'function') this.parent.onThermometer(address,"AMBIENT",temperature);
                debug(sprintf("EXT-AMBIENT %s ADDRESS %s",temperature,address));
            }  

            // new regex:thActuators = RegEx_Create( \"^\*#4\*(\d{1,2})\*19\*(\d)\*(\d)$" );
            extract = frame.match(/^\*#4\*(\d+)\*19\*(\d)\*(\d)$/);
            if (extract) {
                var address = parseInt(extract[1]);
                var cooling = inArray(extract[2],[1,2]);
                var heating = inArray(extract[3],[1,2]);
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onThermostat) == 'function') this.parent.onThermostat(address,"HEATING",heating);
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onThermostat) == 'function') this.parent.onThermostat(address,"COOLING",cooling);
                debug(sprintf("HEATING %s COOLING %s ADDRESS %s",heating,cooling,address));
            }            

            // *#4*<where>#<actuator>*20*<status>##
            extract = frame.match(/^\*#4\*(\d+)#\d\*20\*\d$/);
            if (extract) {
                /* Actuator command, asks for status */
                this.command.send(sprintf("*#4*19*%d##", extract[1]));
            }

            // DRYCONTACT: 	*25*31#[0-1]*3<where>##   -> Close (0 after a state request, 1 after an event) -> TRUE
            // 				*25*32#[0-1]*3<where>##   -> Open  (0 after a state request, 1 after an event) -> FALSE
            extract = frame.match(/^\*25\*3(\d)#[0-1]\*3(\d+)$/);
            if (extract) {
                var value = parseInt(extract[1],10) == 1 ? true : false;
                var address = parseInt(extract[2],10);
                if (this.parent != null && this.parent.onMonitor != null && typeof(this.parent.onContactSensor) == 'function') this.parent.onContactSensor(address,value);
                debug(sprintf("DRYCONTACT %s ADDRESS %s",value,address));
            }   
        }.bind(this));
    }

    getThermostatStatus(_address) {
        this.command.send(sprintf("*#4*%d##*#4*19*%d##",_address,_address));
    }

    getContactState(_address) {
        this.command.send(sprintf("*#25*3%d
