import groovy.json.JsonSlurper
/**
 *  Radio Thermostat local API
 *
 *  Copyright 2014 Dattas Moonchaser
 *
 *  Permission is hereby granted, free of charge, to any person obtaining a copy
 *  of this software and associated documentation files (the "Software"), to deal
 *  in the Software without restriction, including without limitation the rights
 *  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 *  copies of the Software, and to permit persons to whom the Software is
 *  furnished to do so, subject to the following conditions:
 *  
 *  The above copyright notice and this permission notice shall be included in
 *  all copies or substantial portions of the Software.
 *  
 *  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 *  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 *  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 *  AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 *  LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 *  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 *  THE SOFTWARE.
 *
 */
metadata {
	definition (name: "Radio Thermostat local API", namespace: "Thermostat", author: "R K") {
    	capability "Temperature Measurement"
	capability "Thermostat"
        capability "Refresh"
        capability "Sensor"
        capability "Polling"
	}

	simulator {
		// TODO: define status and reply messages here
	}

	tiles {
    	valueTile("temperature", "device.temperature", width: 2, height: 2) {
			state("temperature", label:'${currentValue}°', unit:"F",
				backgroundColors:[
					[value: 31, color: "#153591"],
					[value: 44, color: "#1e9cbb"],
					[value: 59, color: "#90d2a7"],
					[value: 74, color: "#44b621"],
					[value: 84, color: "#f1d801"],
					[value: 95, color: "#d04e00"],
					[value: 96, color: "#bc2323"]
				]
			)
		}
		standardTile("mode", "device.thermostatMode", inactiveLabel: false, decoration: "flat") {
			state "off", label:'${name}', action:"thermostat.setThermostatMode"
			state "cool", label:'${name}', action:"thermostat.setThermostatMode"
			state "heat", label:'${name}', action:"thermostat.setThermostatMode"
			state "emergencyHeat", label:'${name}', action:"thermostat.setThermostatMode"
		}
		standardTile("fanMode", "device.thermostatFanMode", inactiveLabel: false, decoration: "flat") {
			state "fanAuto", label:'${name}', action:"thermostat.setThermostatFanMode"
            state "fanAuto/Circulate", label:'${name}', action:"thermostat.setThermostatFanMode"
			state "fanOn", label:'${name}', action:"thermostat.setThermostatFanMode"
		}
    	controlTile("heatSliderControl", "device.heatingSetpoint", "slider", height: 1, width: 2, inactiveLabel: false) {
			state "setHeatingSetpoint", action:"thermostat.setHeatingSetpoint", backgroundColor:"#d04e00"
		}
		valueTile("heatingSetpoint", "device.heatingSetpoint", inactiveLabel: false, decoration: "flat") {
			state "heat", label:'${currentValue}° heat', unit:"F", backgroundColor:"#ffffff"
		}
		controlTile("coolSliderControl", "device.coolingSetpoint", "slider", height: 1, width: 2, inactiveLabel: false) {
			state "setCoolingSetpoint", action:"thermostat.setCoolingSetpoint", backgroundColor: "#1e9cbb"
		}
		valueTile("coolingSetpoint", "device.coolingSetpoint", inactiveLabel: false, decoration: "flat") {
			state "cool", label:'${currentValue}° cool', unit:"F", backgroundColor:"#ffffff"
		}
		standardTile("refresh", "device.temperature", inactiveLabel: false, decoration: "flat") {
			state "default", action:"refresh.refresh", icon:"st.secondary.refresh"
		}
		main "temperature"
        details(["temperature", "mode", "fanMode", "heatSliderControl", "heatingSetpoint", "coolSliderControl", "coolingSetpoint", "refresh"])
	}
}

// parse events into attributes
def parse(String description) {
	log.debug "Parsing '${description}'"
    def map = [:]
    def retResult = []
    def descMap = parseDescriptionAsMap(description)
    log.debug "parse returns $descMap"
    def body = new String(descMap["body"].decodeBase64())
    def slurper = new JsonSlurper()
    def result = slurper.parseText(body)
    log.debug "json is: $result"
    if (result.containsKey("success")){
    	//Do nothing as nothing can be done. (runIn doesn't appear to work here and apparently you can't make outbound calls here)
        log.debug "returning now"
        return null
    }
    if (result.containsKey("tmode")){
    	def tmode = getModeMap()[result.tmode]
        if(device.currentState("thermostatMode")?.value != tmode){
            retResult << createEvent(name: "thermostatMode", value: tmode)
        }
    }
    if (result.containsKey("fmode")){
    	def fmode = getFanModeMap()[result.fmode]
        if (device.currentState("thermostatFanMode")?.value != fmode){
            retResult << createEvent(name: "thermostatFanMode", value: fmode)
        }
    }
    if (result.containsKey("t_cool")){
    	def t_cool = getTemperature(result.t_cool)
    	if (device.currentState("coolingSetpoint")?.value != t_cool.toString()){
            retResult << createEvent(name: "coolingSetpoint", value: t_cool)
        }
    }
    if (result.containsKey("t_heat")){
    	def t_heat = getTemperature(result.t_heat)
    	if (device.currentState("heatingSetpoint")?.value != t_heat.toString()){
            retResult << createEvent(name: "heatingSetpoint", value: t_heat)
        }
    }
    if (result.containsKey("temp")){
    	def temp = getTemperature(result.temp)
    	if (device.currentState("temperature")?.value != temp.toString()){
            retResult << createEvent(name: "temperature", value: temp)
        }        
    }

    log.debug "Parse returned $retResult"
    if (retResult.size > 0){
		return retResult
    } else {
    	return null
    }

}
def poll() {
	log.debug "Executing 'poll'"
	sendEvent(descriptionText: "poll keep alive", isStateChange: false)  // workaround to keep polling from being shut off
	refresh()
}
def modes() {
	["off", "heat", "cool"]
}
def parseDescriptionAsMap(description) {
	description.split(",").inject([:]) { map, param ->
		def nameAndValue = param.split(":")
		map += [(nameAndValue[0].trim()):nameAndValue[1].trim()]
	}
}

def getModeMap() { [
	0:"off",
	2:"cool",
	1:"heat",
]}

def getFanModeMap() { [
	0:"fanAuto",
	1:"fanAuto/Circulate",
    2:"fanOn"
]}
def getTemperature(value) {
	if(getTemperatureScale() == "C"){
		return (((value-32)*5.0)/9.0)
	} else {
		return value
	}
}
