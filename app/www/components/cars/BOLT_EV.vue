<template></template>
<script>
    import eventBus from './../modules/event.vue';
    import storage from './../modules/storage.vue';

    export default {
        data() {
            return {
                initCMD: [
                    'ATD', 'ATZ', 'ATE0', 'ATL0', 'ATS0', 'ATH1', 'AT0', 'ATSTFF', 'ATFE', 'ATSP6'
                ],
                inStandbyMode: false,
                offset: 0,
                commands: [
                  '228334', //displayed SoC
                  '015B',   //bms SoC
                  '222885', //pack voltage
                  '222414', //pack current
                  '0146',   //ext temp
                  '2241A3', //battery capacity
                  '22434F', //battery temp
                  '22000D', //speed
                  '22436B', //DC charger current
                  '22436C', //DC charger voltage
                ],
                cmd_index: 0,
            };
        },
        methods: {
            init() {
                var self = this;

                // listener to wakeup after standby mode
                eventBus.$off('wakeup');
                eventBus.$on('wakeup', () => self.inStandbyMode = false);

                // unsubscribe from prior existing listener
                bluetoothSerial.unsubscribe();

                // subscribe to data
                bluetoothSerial.subscribe('>', data => {
                    if (self.inStandbyMode) return;
                    // remove spaces
                    data = data.trim().replace(/\s/g, '');
                    console.log({
                        data
                    });
                    // send debug data to backend if debug mode enabled
                    if (DEBUG) Vue.http.post(RESTURL + 'debug', {
                        data,
                        akey: storage.getValue('akey')
                    });
                    // error and empty response detection to start re-initialization
                    if (data.indexOf('CANERROR') !== -1 ||
                        data.indexOf('STOPPED') !== -1 ||
                        data.indexOf('UNABLETOCONNECT') !== -1 ||
                        data.indexOf('BUFFERFULL') !== -1) {
                        // there was an error - reset offset, to start with first command afterwards
                        self.offset = -1;
                        // emit obd2 error
                        eventBus.$emit('obd2Error', data);
                    }
                    if (self.offset + 1 === self.initCMD.length) {
                        // init of dongle finished, parse data and just send the OBD2 command
                        eventBus.$emit('obd2Data', self.parseData(data));
                        setTimeout(function(){
                          bluetoothSerial.write(self.commands[self.cmd_index] + '\r');
                          self.cmd_index = (self.cmd_index >= self.commands.length - 1) ? 0 : self.cmd_index+1;
                        }, 2000/self.commands.length); //Send commands faster to make it through longer command list in 2 seconds.
                    } else bluetoothSerial.write(self.initCMD[++self.offset] + '\r');
                }, err => console.error(err));

                // initialize the dongle by sending the first command
                bluetoothSerial.write(self.initCMD[self.offset] + '\r');
            },
            parseData(data) {
                var self = this,
                    parsedData = {},
                    baseData = self.getBaseData();

                try {
                  // '228334', //displayed SoC
                  // '222885', //volts
                  // '222414', //amps
                  // '0146',   //ext temp
                  // '2241A3', //battery capacity
                  // '22434F', //battery temp
                  // '22000D', //speed
                    if (data.indexOf('228334') !== -1) {
                        parsedData = {
                            SOC_DISPLAY: (parseInt(data.substr(data.indexOf('228334'), 6).slice(-2), 16) * 100 / 255).toFixed(2), // following 2 bytes from response after 415B
                        }
                    } //TODO parse each response correctly
                      // CHARGING: if charge current is > 0?
                      // SPEED(obd): A
                      // SOC_BMS: A*100/255
                      // BATTERY_MIN_TEMPERATURE: A-40
                      // BATTERY_MAX_TEMPERATURE:
                      // DC_BATTERY_VOLTAGE: ((A*256)+B)/2 ((
                      //         parseInt(
                      //             extractedSecondData.slice(2, 4), 16 // second byte within 2nd block
                      //         ) << 8) +
                      //     parseInt(
                      //         extractedSecondData.slice(4, 6), 16 // third byte within 2nd block
                      //     )
                      // ) / 10,
                      // DC_BATTERY_CURRENT: ((Signed(A)*256+b))/20 helper.parseSigned(
                      //   (extractedFirstData.slice(12, 14) + extractedSecondData.slice(0, 2)), 16 // concat 7th byte of first block with first byte of second block
                      // ) * 0.1,
                      // SOC_DISPLAY: A*100/255
                      // SOH: ((A*256)+B)/30 = out of 60kWh ((
                      //         parseInt(
                      //             extractedFourthData.slice(0, 2), 16 // first byte within 4th block
                      //         ) << 8) +
                      //     parseInt(
                      //         extractedFourthData.slice(2, 4), 16 // second byte within 4th block
                      //     )
                      // ) / 10
                } catch (err) {
                    console.error(err);
                }
                // extend with base data
                Object.keys(baseData).forEach(key => parsedData[key] = baseData[key]);
                console.log({
                    parsedData
                });
                return parsedData;
            },
            getBaseData() {
                return {
                    CAPACITY: 60,
                    SLOW_SPEED: 1.44,
                    NORMAL_SPEED: 7.68,
                    FAST_SPEED: 56
                };
            },
            standbyMode() {
                var self = this;

                self.inStandbyMode = true;
                // unsubscribe, and emit low power mode command
                bluetoothSerial.unsubscribe();
                bluetoothSerial.write('ATLP\r', () => {
                    eventBus.$emit('standby');
                }, err => eventBus.$emit('standby', err));
            }
        }
    }
</script>