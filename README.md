This flow connects to the Volkszaehler push-server via a websocket-node and receives json formatted measurements.
These measurements are then transformed in a function-node to be send to influxdb's telegraf via the mqtt protocol.

To setup the flow edit the 'uuidMap' object in the function node:
-	Replace the XXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX with the actual UUIDs from Volkszaehler
-	Set the matching mqtt topic in topic
-	Set the influxdb measurement in measurement
-	Provide optional any number of tags to be able to filter data in telegraf or influxdb
