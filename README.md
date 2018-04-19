# Myhome
Legrand Myhome automation 
{
    "platforms": [{
        "platform": "MyHome Gateway",
        "ipaddress": "192.168.1.1",
        "password": "12345",
        "discovery": false,
        "devices": [
                /*Static list of devices*/
            ]
        }], 
    "bridge": {
        "username": "CC:22:3D:E3:CE:31", 
        "name": "MyHome HomeBridge Adapter", 
        "pin": "342-52-220", 
        "port": 51826
    }, 
    "description": "My Fantastic Legrand MyHome System", 
    "accessories": []
}
