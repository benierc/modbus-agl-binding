# Mosbus Binding

Modbus binding support TCP Modbus with format convertion for multi-register type as int32, Float, ...

## Dependencies:
 * AGL application framework 'agl-app-framework-binder-devel'
 * AGL controller 'agl-libappcontroller-devel'
 * AGL helpers 'agl-libafb-helpers-devel'
 * AGL cmake template 'agl-cmake-apps-module'
 * Libmodbus
 * Lua

## AGL dependencies:
 * Declare AGL repository: [(see doc)](https://docs.automotivelinux.org/docs/en/guppy/devguides/reference/2-download-packages.html#install-the-repository)
 * Install AGL controller: [(see doc)](https://docs.automotivelinux.org/docs/en/guppy/devguides/reference/ctrler/controller.html)
 * Install LibModbus

 ```
  # if you did not logout, don't forget to source AGL environement
  source /etc/profile.d/agl-app-framework-binder.sh
 ```

    + download from https://libmodbus.org/download/
    + cd libmodbus-3.1.6/
    + ./configure --prefix=/opt/libmodbus-3.1.6
    + make && sudo make install-strip
    + Update conf.d/00-????-config.cmake with choosen installation directory. ex: set(ENV{PKG_CONFIG_PATH} "$ENV{PKG_CONFIG_PATH}:/opt/libmodbus-3.1.6/lib64/pkgconfig")

## Modbus Binding build
    * mkdir build && cd build
    * cmake ..
    * make

## Running/Testing

By default Kingpigeon devices use a fix IP addr (192.168.1.110). You may need to add this network to your own desktop config before starting your test.
```
sudo ip a add  192.168.1.1/24 dev eth0 # eth0 or whatever is your ethernet card name
ping 192.168.1.110 # check you can ping your TCP modbus device
check default config with browser at http://192.168.1.110
```

### Start Sample Binder
```
afb-daemon --name=afb-kingpigeon --port=1234  --ldpaths=src --workdir=. --roothttp=../htdocs --token= --verbose
open binding UI with browser at localhost:1234
```

### Adding your own config

Json config file is selected from *afb-daemon --name=afb-midlename-xxx* option. This allows you to switch from one json config to the other without editing any file. *'middlename'* is use to select a specific config. As example *--name='afb-mt110@lorient-modbus'* will select *modbus-mt110@lorient-config.json*.

You may also choose to force your config file by exporting CONTROL_CONFIG_PATH environement variable. For further information, check AGL controller documentation [here](https://docs.automotivelinux.org/docs/en/guppy/devguides/reference/ctrler/controllerConfig.html)

**Warning:** some TCP Modbus device as KingPigeon check SalveID even for buildin I/O. Generic config make the assumption that your slaveID is set to *'1'*. 

 
```
export CONTROL_CONFIG_PATH="$HOME/my-agl-config"
afb-daemon --name=afb-myconfig --port=1234  --ldpaths=src --workdir=. --roothttp=../htdocs --token= --verbose
```


## Modbus binding support a set of default encoder for values store within multiple registries. 

    * int16, bool => 1 register 
    * int32 => 2 registers
    * int64 => 4 registers
    * float, floatabcd, floatdabc, ...

Nevertheless user may also add its own encoding/decoding format to handle device specific representation (ex: device info string),or custom application encoding (ex: float to uint16 for an analog output). Custom encoder/decoder are store within user plugin (see sample at src/plugins/kingpigeon).

## API usage:

Modbus binding create one api/verb by sensor. By default each sensor api/verb is prefixed by the RTU uid. With following config mak
```
    "modbus": [
      {
        "uid": "King-Pigeon-MT110",
        "info": "King Pigeon TCP I/O Module",
        "uri" : "tcp://192.168.1.110:502",
        "privilege": "global RTU requiered privilege",
        "autostart" : 1,  // connect to RTU at binder start
        "prefix": "mt110  // api verb prefix 
        "timeout": xxxx // optionnal response timeout in ms
        "debug": 0-3 // option libmodbus debug level
        "hertz": 10  // default pooling for event subscription 
        "iddle": 0   // force event even when value does not change every hertz*iddle count
        "sensors": [
          {
            "uid": "PRODUCT_INFO",
            "info" : "Array with Product Number, Lot, Serial, OnlineTime, Hardware, Firmware",
            "function": "Input_Register",
            "format" : "plugin://king-pigeon/devinfo",
            "register" : 26,
            "privilege": "optionnal sensor required privilege"
          },
          {
            "uid": "DIN01_switch",
            "function": "Coil_input",
            "format" : "BOOL",
            "register" : 1,
            "privilege": "optionnal sensor required privilege"
            "herz": xxx // special pooling rate for this sensor 
            "iddle": xxx // special iddle force event when value does not change 
          },
          {
            "uid": "DIN01_counter",
            "function": "Register_Holding",
            "format" : "UINT32",
            "register" : 6,
            "privilege": "optionnal sensor required privilege"
            "herz": xxx // special pooling rate for this sensor 
          },
    ...      
```

## Modbus controller exposed

### Two builtin api/verb 

  * api://modbus/ping // check if binder is alive
  * api://modbus/info // return registered MTU

### One instropection api/verb per declared RTU
    * api://modbus/mt110/info

### On action api/verb per declared Sensor    
    * api://modbus/mt110/din01_switch
    * api://modbus/mt110/din01_counter
    * etc ...

### For each sensors the API accept 3 actions
    * action=read (return register(s) value after format decoding)
    * action=write (push value on register(s) after format encoding)
    * action=subscribe (sucribe to sensors value changes, frequency is defined by sensor or gloablly at RTU level)

