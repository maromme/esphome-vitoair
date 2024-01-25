# esphome-vitoair
CAN-bus connection to Viessmann Vitoair using ESPhome

## credits:
https://github.com/open3e/open3e/discussions/29#discussioncomment-8117128
https://github.com/open3e/open3e/issues/3
Thanks for the great projekt and the help!

## Resources:
understanding the can-bus protocoll: https://www.csselectronics.com/pages/uds-protocol-tutorial-unified-diagnostic-services

datapoints of the vitoair: https://github.com/open3e/open3e/blob/master/Open3EdatapointsVair.py

description of the datapoints: https://github.com/open3e/open3e/blob/master/Open3Edatapoints.py

## hardware
I used that hardwaresetup:
https://esphome.io/_images/canbus_mcp2515_txs0108e.png

## to be aware of
Usually you should use the can bus in a way that yous send a request - wait for the answer - process the answer and then start over again. I didn't do so. I send requests every x seconds (interval) and hope that in the time between two requests the answer is recieved and processed. I don't take care of multiframe meassages. A interval of 2 seconds seems relaiable in my setting.

For now the setup is just "read-only". It's possible to write values, but for that some more code is necessary.

## configuration
The sample conig contains 4 sensors. Others might follow. In the datapoints.txt you find a (maybe) complete list of the merged files from the open3e project.
To add more datapoints you have to look at three parts of the conig: 
1) the array "dids" under globals
2) the switch statement under canbus
3) add more sensors at the end
   
```
spi:
  id: McpSpi
  clk_pin: GPIO16
  mosi_pin: GPIO5
  miso_pin: GPIO4

globals:
  - id: dids
    type: std::vector<std::vector<std::string>>
    initial_value: '{
      {"327", "OutdoorAirTemperatureSensor", {0x03, 0x22, 0x01, 0x47, 0x00, 0x00, 0x00, 0x00}},
      {"328", "SupplyAirTemperatureSensor",  {0x03, 0x22, 0x01, 0x48, 0x00, 0x00, 0x00, 0x00}},
      {"329", "ExtractAirTemperatureSensor", {0x03, 0x22, 0x01, 0x49, 0x00, 0x00, 0x00, 0x00}},
      {"330", "ExhaustAirTemperatureSensor", {0x03, 0x22, 0x01, 0x4A, 0x00, 0x00, 0x00, 0x00}}
      }'

  - id: loop_dids
    type: int
    restore_value: no
    initial_value: '0' 

canbus:
  - platform: mcp2515
    id: cb
    spi_id: McpSpi
    cs_pin: GPIO14
    can_id: 680
    bit_rate: 250kbps
    on_frame:
     - can_id: 0x690 
       then:
          - lambda: |-
              if(x[0]==0x10 and x[1]==0x0C and x[2]==0x62) {
                  float temp =  float(int16_t((x[5])+((x[6])<<8)))*0.1;
                  int did = int16_t((x[4])+((x[3])<<8));
                  //ESP_LOGD("vitoair", "%d %4.1f", did, temp);
                  switch(did) {
                      case 327:
                        id(va327).publish_state(temp);
                        break;
                      case 328:
                        id(va328).publish_state(temp);
                        break;
                      case 329:
                        id(va329).publish_state(temp);
                        break;
                      case 330:
                        id(va330).publish_state(temp);
                        break;
                    }
                }

interval:
- interval: 2s
  then:
    - lambda: |-
        int i = id(loop_dids);
        std::vector< uint8_t > data0{ id(dids)[i][2][0], id(dids)[i][2][1], id(dids)[i][2][2], id(dids)[i][2][3], id(dids)[i][2][4], id(dids)[i][2][5], id(dids)[i][2][6], id(dids)[i][2][7] };
        id(cb)->send_data(0x680, 0, data0);
        if ( i < 3) { id(loop_dids) = i + 1;}
          else { id(loop_dids) = 0; }

sensor:
  - platform: template
    name: "Outdoor Air Temperature Sensor"
    id: va327
    unit_of_measurement: °C
    device_class: temperature
    state_class: "measurement"
    accuracy_decimals: 1

  - platform: template
    name: "Supply Air Temperature Sensor"
    id: va328
    unit_of_measurement: °C
    device_class: temperature
    state_class: "measurement"
    accuracy_decimals: 1

  - platform: template
    name: "Extract Air Temperature Sensor"
    id: va329
    unit_of_measurement: °C
    device_class: temperature
    state_class: "measurement"
    accuracy_decimals: 1

  - platform: template
    name: "Exhaust Air Temperature Sensor"
    id: va330
    unit_of_measurement: °C
    device_class: temperature
    state_class: "measurement"
    accuracy_decimals: 1

```
