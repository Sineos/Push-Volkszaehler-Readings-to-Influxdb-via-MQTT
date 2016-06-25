
This flow connects to the Volkszaehler push-server via a websocket-node and receives json formatted measurements.
These measurements are then transformed in a function-node to be send to influxdb's telegraf via the mqtt protocol.

#How To
##Prerequisites

 - [NodeRed](http://nodered.org/)
 - [Vzlogger](https://github.com/volkszaehler/vzlogger)
 - [Volkszaehler.org](https://github.com/volkszaehler/volkszaehler.org)
 - [Mosquitto](http://mosquitto.org/) or any other MQTT message broker
 - [Telegraf](https://influxdata.com/time-series-platform/telegraf/)
 - [InfluxDB](https://influxdata.com/time-series-platform/influxdb/)
 - [Grafana](http://grafana.org/) Optional for visualization

Setup instructions are widely available in the WWW for these products and may vary according to the hardware (Raspberry PI, i386 etc) and linux distribution.

##Configuration

###Vzlogger
Make sure you Vzlogger installation past the 28 Jun 2015 to support the push settings.

*vzlogger.conf*

         // realtime notification settings
        "push": [
            {
                "url": "http://127.0.0.1:5582"  // notification destination, e.g. frontend push-server
            }
    ],

If your volkszaehler.org instance is running on the same machine as vzlogger you can leave the `url` setting, otherwise replace it with the IP of your volkszaehler.org instance.

###volkszaehler.org
To use the volkszaehler push feature you will need a volkszaehler version after this [commit](https://github.com/volkszaehler/volkszaehler.org/commit/b415699b5ec281526791ad80dec9b7abf64faee8). I recommend the manual installation of volkszaehler, described [here](http://wiki.volkszaehler.org/software/middleware/installation) in the chapter "Manuelle Installation". This makes updating the installation a lot easier later.

volkszaehler.conf.php

    /**
     * Push server settings
     */
    $config['push']['enabled'] = true;		// set to true to enable push updates
    $config['push']['server'] = 5582;		// vzlogger will push to this ports (binds on 0.0.0.0)
    $config['push']['broadcast'] = 8082;	// frontend will subscribe on this port (binds on 0.0.0.0)
    $config['push']['routes']['wamp'] = array('/', '/ws');		// routes for wamp access
    $config['push']['routes']['websocket'] = array('/socket');			// routes for plain web sockets, try array('/socket')

Modify `$config['push']['enabled']` and `$config['push']['routes']['websocket']` as shown above.

Change into the volkszaehler.org path and start the push server with:

     php misc/tools/push-server.php

Alternatively you can setup a service to have it started at boot:

    sudo nano /etc/systemd/system/push-server.service

paste following template to the file:

    [Unit]
    Description=push-server
    After=syslog.target network.target
    Requires=
    
    [Service]
    ExecStart=/usr/bin/php /var/www/volkszaehler.org/misc/tools/push-server.php
    ExecReload=/bin/kill -HUP $MAINPID
    StandardOutput=null
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
Be sure to have right path to the push-server.php in ExecStart.

To start the push-server use:

     sudo systemctl start push-server

To enable the push-server at boot:

    sudo systemctl enable push-server

###NodeRed
####Import Flow
Copy the flow.json in [raw format](https://raw.githubusercontent.com/Sineos/Push-Volkszaehler-Readings-to-Influxdb-via-MQTT/master/flow.json) to your clipboard.
In NodeRed import the copied flow:
![import_flow](https://raw.githubusercontent.com/Sineos/Push-Volkszaehler-Readings-to-Influxdb-via-MQTT/master/src_readme/import_nodered.jpg)

####Configure push-server input
![enter image description here](https://raw.githubusercontent.com/Sineos/Push-Volkszaehler-Readings-to-Influxdb-via-MQTT/master/src_readme/edit_websocket.jpg)
Add / modify the URL to point to your push-server IP. If NodeRed and the volkszaehler / push-server is running on the same machine use:

    ws://127.0.0.1:8082/socket

####Configure function node
Double-click the function node `Format payload for influxdb`
![enter image description here](https://raw.githubusercontent.com/Sineos/Push-Volkszaehler-Readings-to-Influxdb-via-MQTT/master/src_readme/edit_function.jpg)

 - Replace the `XXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX` with the actual UUIDs from your volkszaehler.org / vzlogger installation
 - The `topic` is needed for the MQTT message broker. This reflect the "path" or "channel" under which the MQTT message broker will publish the data. This will be the channel that we use in Telegraf to collect the data
 - `Measurement` is needed to put the data into the InfluxDB and will be used along with the `tags`. Read the InfluxDB manual [here](https://docs.influxdata.com/influxdb/v0.13/concepts/key_concepts/) and [here](https://docs.influxdata.com/influxdb/v0.13/concepts/schema_and_data_layout/) for a better understanding of the measurement and tag concept.
 - `Tags` are to be used alongside the `measurement` and need to be specified in the format `Tag_Key = Tag_Value`. They can be used to sort, query and cluster data within InfluxDB. You can specify none to an arbitrary number of tags.

Within uuidMap you can specify all channels from volkszaehler. It is important to keep the following structure intact:

    'XXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX':{
        topic:'/power/sml/Bezug',
    	measurement:'power_sml',
    	tags:{
    		tag1:'Type=sml',
    		tag2:'Location=Bezug'
    		tag3:'bla'
    	}
    }, 

####Configure MQTT output
![enter image description here](https://raw.githubusercontent.com/Sineos/Push-Volkszaehler-Readings-to-Influxdb-via-MQTT/master/src_readme/edit_mqtt.jpg)
Edit the `Server` to match the IP and port of the MQTT message broker (e.g. Mosquitto).

###Telegraf
In the telegraf.conf file you have to specify an Input-Plugin for the MQTT protocol:

    # Read metrics from MQTT topic(s)
    [[inputs.mqtt_consumer]]
      servers = ["localhost:1883"]
      ## MQTT QoS, must be 0, 1, or 2
      qos = 0
    
      ## Topics to subscribe to
      topics = [
        "/power/#",
      ]
    
      # if true, messages that can't be delivered while the subscriber is offline
      # will be delivered when it comes back (such as on service restart).
      # NOTE: if true, client_id MUST be set
      persistent_session = false
      # If empty, a random client ID will be generated.
      client_id = ""
    
      ## username and password to connect MQTT server.
      # username = "telegraf"
      # password = "metricsmetricsmetricsmetrics"
    
      ## Optional SSL Config
      # ssl_ca = "/etc/telegraf/ca.pem"
      # ssl_cert = "/etc/telegraf/cert.pem"
      # ssl_key = "/etc/telegraf/key.pem"
      ## Use SSL but skip chain & host verification
      # insecure_skip_verify = false
    
      ## Data format to consume.
      ## Each data format has it's own unique set of configuration options, read
      ## more about them here:
      ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
      data_format = "influx"

 - Modify `servers` to match the IP and port of the MQTT message broker (e.g. Mosquitto) 
 - Modiy `topics` to the topic(s) you used in the function node of the flow. Note that the `#` will match anything after the first part of the path
