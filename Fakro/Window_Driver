/**
 *  Fakro Roof Window driver for Hubitat
 * 
 *  Copyright 2021 Simon J Burke - ScruffySJB
 *
 *  Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except
 *  in compliance with the License. You may obtain a copy of the License at:
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software distributed under the License is distributed
 *  on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License
 *  for the specific language governing permissions and limitations under the License.
 *  
 * 
 ***************************Version 1.0******************************************
 *  Version History
 *
 *  2021-09-19: Version 1.0 Initial release
 *  2023-05-28: Version 1.6 - added automatic refresh after a command is sent and set preferences after configure is called
 *
*/

import groovy.transform.Field

metadata {
   definition (name: "Fakro Window Driver", namespace: "Scruffy-sjb", author: "Scruffy-SJB") {
      capability "Actuator"
      capability "WindowShade" 
      capability "Switch"  
      capability "Refresh"
      capability "Configuration"
	  capability "Notification"
	  capability "Sensor"
	  capability "Health Check"
	  
	  command "open"
      command "close"
      command "off"
      command "on"

      attribute "windowOpen", "String"
	  attribute "refreshRate", "String"
		
      fingerprint  mfr: "0085", deviceType: "0002", deviceId: "0017", inClusters: "0x5E,0x86,0x72,0x5A,0x85,0x59,0x73,0x25,0x26,0x27,0x70,0x71,0x77", deviceJoinName: "Fakro FTP-V, FTU-V Roof Window" 
   }

// Chain Execution Speeds
def chain =[:]
    chain << ["1" : "Slowest"]
    chain << ["2" : "Medium Slow (Default)"]
    chain << ["3" : "Medium Fast"]
    chain << ["4" : "Fastest"]
    
// Chain Execution Speeds (Rain Sensor)
def rain =[:]
    rain << ["1" : "Slowest"]
    rain << ["2" : "Medium Slow (Default)"]
    rain << ["3" : "Medium Fast"]
    rain << ["4" : "Fastest"]
    
// Calibration
def calib =[:]
    calib << ["1" : "Calibration (Default)"]
    calib << ["2" : "Decalibration"]

// Goto Position
def gotop =[:]
    gotop << ["1" : "Open to Maximum (Default)"]
    gotop << ["2" : "Goto Previous Position"]

//Rates for Poll
def rates = [:]
    rates << ["0" : "Disabled"]
    rates << ["1" : "Refresh every minute"]
    rates << ["5" : "Refresh every 5 minutes (Default)"]
    rates << ["10" : "Refresh every 10 minutes"]
    rates << ["15" : "Refresh every 15 minutes"]
    
@Field static Map<String, Map<Short, String>> supervisedPackets = [:]
@Field static Map<String, Short> sessionIDs = [:]   
@Field static Map commandClassVersions = 
[
         0x01: 2    // z-wave+ info
        ,0x20: 1    // basic
        ,0x25: 1    // switch binary
        ,0x26: 3    // switchMultiLevel    
        ,0x27: 1    // switch all
        ,0x59: 1    // association group info
        ,0x5A: 1    // device reset locally
        ,0x70: 1    // configuration    
        ,0x71: 3    // notification
        ,0x72: 2    // manufacturer specific
        ,0x73: 1    // powerlevel    
        ,0x77: 1    // node naming
        ,0x85: 2    // association
     	,0x86: 2    // version
        ,0x98: 1    // security
]   
    
   preferences {
      //parameter 7
      input name: "chainExecutionSpeed", type: "enum", options: chain, description: "Select chain execution speed", title: "Window Execution Speed", range: 1..4, defaultValue: 2, required: true
      //parameter 8
      input name: "rainExecutionSpeed", type: "enum", options: rain, description: "Select chain execution speed (rain)", title: "Window Execution Speed (Rain)", range: 1..4, defaultValue: 2, required: true       
	  //parameter 12
      input name: "calibration", type: "enum", options: calib, description: "Select calibration", title: "Calibration", range: 1..2, defaultValue: 1, required: true
      //parameter 13	  
      input name: "gotoPosition", type: "enum", options: gotop, description: "Select position", title: "Go to Position", range: 1..2, defaultValue: 1, required: true
      //parameter 15	  
      input name: "ventilation", type: "number", description: "", title: "Close after set time (Minutes)", range: "0..120", defaultValue: 0, required: false
      input name: "refreshRate", type: "enum", title: "Refresh rate", options: rates, description: "Select refresh rate", defaultValue: "5", required: false
   }
}

// parse events into attributes
def parse(String description) {
    // log.debug "Parsing '${description}'"
    def result = []
    if (description.startsWith("Err 106")) {
        state.sec = 0
        result = createEvent(descriptionText: description, isStateChange: true)
    }
    else {
        def cmd = zwave.parse(description,[0x75:1])
        if (cmd) {
            result += zwaveEvent(cmd)
            //				log.debug "Parsed ${cmd} to ${result.inspect()}"
        } else {
            log.debug "Non-parsed event: ${description}"
        }
    }
    return result
}

def zwaveEvent(hubitat.zwave.commands.switchmultilevelv3.SwitchMultilevelReport cmd){
//    log.info "Report Received : Window open ${cmd.value}%"
    sendEvent(name: "lastCheckin", value: new Date())
	windowEvents(cmd)
}

def zwaveEvent(hubitat.zwave.commands.basicv1.BasicReport cmd) {
//    log.debug "Basic report received - cmd.value = $cmd.value"
    sendEvent(name: "lastCheckin", value: new Date())
    windowEvents(cmd)    
}

private void windowEvents(hubitat.zwave.Command cmd) {
   def position = cmd.value

   if (position > 0)  {
      sendEvent(name: "windowOpen", value: "yes")
      switchValue = "on" 
      log.info "$device.displayName window is open - switch is $switchValue - position is $position"       
   } else {
      sendEvent(name: "windowOpen", value: "no")
      switchValue = "off"  
      log.info "$device.displayName window is closed - switch is $switchValue - position is $position"       
   }
    
    sendEvent(name: "position", value: position, unit: "%")  
    sendEvent(name: "switch", value: switchValue)
}

def zwaveEvent(hubitat.zwave.commands.associationv2.AssociationReport cmd) {
    def result = []
    if (cmd.nodeId.any { it == zwaveHubNodeId }) {
        result << sendEvent(descriptionText: "$device.displayName is associated in group ${cmd.groupingIdentifier}")
    } else if (cmd.groupingIdentifier == 1) {
        // We're not associated properly to group 1, set association
        result << sendEvent(descriptionText: "Associating $device.displayName in group ${cmd.groupingIdentifier}")
        result << response(zwave.associationV1.associationSet(groupingIdentifier:cmd.groupingIdentifier, nodeId:zwaveHubNodeId))
    }
    log.info "Report Received : $cmd"
    result
}

def zwaveEvent(hubitat.zwave.commands.notificationv3.NotificationReport cmd) {
    def events = []
    switch (cmd.notificationType) {
//       case 8:
//        processPowerMgtNot(cmd)
//        break

//        case 9:
//        processSystemNot(cmd)
//        break
   }

    log.info "Notification Report : $cmd"
    sendEvent(name: "lastCheckin", value: new Date())
}

def zwaveEvent(hubitat.zwave.commands.securityv1.SecurityMessageEncapsulation	 cmd) { // Devices that support the Security command class can send messages in an encrypted form; they arrive wrapped in a SecurityMessageEncapsulation command and must be unencapsulated
    log.debug "raw secEncap $cmd"
    state.sec = 1
    def encapsulatedCommand = cmd.encapsulatedCommand ([0x20: 1, 0x80: 1, 0x70: 1, 0x72: 1, 0x31: 5, 0x26: 3, 0x75: 1, 0x40: 2, 0x43: 2, 0x86: 1, 0x71: 3, 0x98: 2, 0x7A: 1 ]) 

    if (encapsulatedCommand) {
        log.warn "Encapsulated command extracted sucessfully"
        return zwaveEvent(encapsulatedCommand)
    } else {
        log.warn "Unable to extract encapsulated cmd from $cmd"
        createEvent(descriptionText: cmd.toString())
    }
}

def zwaveEvent(hubitat.zwave.Command cmd) {
    def map = [ descriptionText: "${device.displayName}: ${cmd}" ]
    log.warn "mics zwave.Command - ${device.displayName} - $cmd"
    sendEvent(map)
}

def zwaveEvent(hubitat.zwave.commands.manufacturerspecificv2.ManufacturerSpecificReport cmd) {
    if (cmd.manufacturerName) { updateDataValue("manufacturer", cmd.manufacturerName) }
    if (cmd.productTypeId) { updateDataValue("productTypeId", cmd.productTypeId.toString()) }
    if (cmd.productId) { updateDataValue("productId", cmd.productId.toString()) }
    if (cmd.manufacturerId){ updateDataValue("manufacturerId", cmd.manufacturerId.toString()) }
    log.info "Report Received : $cmd"
}

def zwaveEvent(hubitat.zwave.commands.zwaveplusinfov2.ZwaveplusInfoReport cmd) {
    if (cmd.zWaveplusNodeType) { updateDataValue("Z-Wave+NodeType", cmd.zWaveplusNodeType.toString()) }
    if (cmd.zWaveplusRoleType) { updateDataValue("Z-Wave+RoleType", cmd.zWaveplusRoleType.toString()) }
    if (cmd.zWaveplusVersion){ updateDataValue("Z-Wave+Version", cmd.zWaveplusVersion.toString()) }
    log.info "Report Received : $cmd"
}

def zwaveEvent(hubitat.zwave.commands.configurationv2.ConfigurationReport cmd ) {
//    log.info "Configuration Report Received : $cmd"
    def events = []

    switch (cmd.parameterNumber) {
        case 7:
        events << processParam7(cmd)
        break

        case 8:
        events << processParam8(cmd)
        break
        
        case 12:
        events << processParam12(cmd)
        break

        case 13:
        events << processParam13(cmd)
        break

        case 15:
        events << processParam15(cmd)
        break
		
		default:
		log.debug "Parameter $cmd received"
		break
    }
}

def zwaveEvent(hubitat.zwave.commands.versionv1.VersionReport cmd) {
    log.info "Version Report: $cmd"
    def zWaveLibraryTypeDisp  = String.format("%02X",cmd.zWaveLibraryType)
    def zWaveLibraryTypeDesc  = ""
    switch(cmd.zWaveLibraryType) {
        case 1:
        zWaveLibraryTypeDesc = "Static Controller"
        break

        case 2:
        zWaveLibraryTypeDesc = "Controller"
        break

        case 3:
        zWaveLibraryTypeDesc = "Enhanced Slave"
        break

        case 4:
        zWaveLibraryTypeDesc = "Slave"
        break

        case 5:
        zWaveLibraryTypeDesc = "Installer"
        break

        case 6:
        zWaveLibraryTypeDesc = "Routing Slave"
        break

        case 7:
        zWaveLibraryTypeDesc = "Bridge Controller"
        break

        case 8:
        zWaveLibraryTypeDesc = "Device Under Test (DUT)"
        break

        case 0x0A:
        zWaveLibraryTypeDesc = "AV Remote"
        break

        case 0x0B:
        zWaveLibraryTypeDesc = "AV Device"
        break

        default:
            zWaveLibraryTypeDesc = "N/A"
    }

    def zWaveVersion = String.format("%d.%02d",cmd.zWaveProtocolVersion,cmd.zWaveProtocolSubVersion)
    def firmwareLevel = String.format("%d.%02d",cmd.firmware0Version,cmd.firmware0SubVersion)
    log.info "Z-Wave type is $zWaveLibraryTypeDesc - Z-Wave Version is $zWaveVersion - Device Firmware Level is $firmwareLevel"
}   

def configure() {
    state.clear()            // Clear all previous states
    unschedule()      
    def cmds = []
    cmds << zwave.configurationV1.configurationSet(configurationValue: chainExecutionSpeed == "1" ? [0x01] : chainExecutionSpeed == "2" ? [0x02] : chainExecutionSpeed == "3" ? [0x03] : chainExecutionSpeed == "4" ? [0x04] : [0x02], parameterNumber:7, size:1, scaledConfigurationValue:  chainExecutionSpeed == "1" ? 0x01 : chainExecutionSpeed == "2" ? 0x02 : chainExecutionSpeed == "3" ? 0x03 : chainExecutionSpeed == "4" ? 0x04 : 0x02)
    cmds << zwave.configurationV1.configurationSet(configurationValue: chainExecutionSpeed == "1" ? [0x01] : rainExecutionSpeed == "2" ? [0x02] : rainExecutionSpeed == "3" ? [0x03] : rainExecutionSpeed == "4" ? [0x04] : [0x02], parameterNumber:8, size:1, scaledConfigurationValue:  rainExecutionSpeed == "1" ? 0x01 : rainExecutionSpeed == "2" ? 0x02 : rainExecutionSpeed == "3" ? 0x03 : rainExecutionSpeed == "4" ? 0x04 : 0x02)
    cmds << zwave.configurationV1.configurationSet(configurationValue: calibration == "1" ? [0x01] : calibration == "2" ? [0x02] : [0x01], parameterNumber:12, size:1, scaledConfigurationValue:  calibration == "1" ? 0x01 : calibration == "2" ? 0x02 : 0x01)
    cmds << zwave.configurationV1.configurationSet(configurationValue: gotoPosition == "1" ? [0x01] : gotoPosition == "2" ? [0x02] : [0x01], parameterNumber:13, size:1, scaledConfigurationValue:  gotoPosition == "1" ? 0x01 : gotoPosition == "2" ? 0x02 : 0x01)
    cmds << zwave.configurationV1.configurationSet(parameterNumber:15, scaledConfigurationValue: ventilation.toInteger())
    cmds << zwave.sensorMultilevelV5.sensorMultilevelGet(sensorType:1, scale:1) 
    cmds << zwave.switchMultilevelV3.switchMultilevelGet()
    cmds << zwave.configurationV1.configurationGet(parameterNumber:7)
    cmds << zwave.configurationV1.configurationGet(parameterNumber:8)    
    cmds << zwave.configurationV1.configurationGet(parameterNumber:12)
    cmds << zwave.configurationV1.configurationGet(parameterNumber:13)
    cmds << zwave.configurationV1.configurationGet(parameterNumber:15)
    cmds << zwave.manufacturerSpecificV1.manufacturerSpecificGet()
    cmds << zwave.versionV1.versionGet()
    cmds << zwave.zwaveplusInfoV2.zwaveplusInfoGet()
    
    sendEvent(name: "configuration", value: "sent", displayed: true)
    log.info "Configuration sent"
    secureSequence(cmds)    
    updated()
}

def updated() {
    if (!state.updatedLastRanAt || new Date().time >= state.updatedLastRanAt + 2000) {
        state.updatedLastRanAt = new Date().time
        unschedule(refresh)
        unschedule(poll)
        sendEvent(name: "refreshRate", value: refreshRate)
		
		switch(refreshRate) {
            case "1":
            runEvery1Minute(poll)
            log.info "Refresh Scheduled for every minute"
            break
            case "15":
            runEvery15Minutes(poll)
            log.info "Refresh Scheduled for every 15 minutes"
            break
            case "10":
            runEvery10Minutes(poll)
            log.info "Refresh Scheduled for every 10 minutes"
            break
            case "5":
            runEvery5Minutes(poll)
            log.info "Refresh Scheduled for every 5 minutes"
            break
            case "0":
            runIn(60*60*24, 'poll')    // Must poll at least once per day
            log.info "Refresh off"}
    }
    else {
        log.warn "update ran within the last 2 seconds"
    }
}

// PING is used by Device-Watch in attempt to reach the Device
def ping() {
    refresh()
}

//Refresh
def refresh() {
    def cmds = []
    log.trace "$device.displayName Refresh called"
    cmds <<	zwave.basicV1.basicGet()						// get mode (basic)	
//    cmds <<	zwave.sensorMultilevelV1.sensorMultilevelGet()	// get temp
    cmds << zwave.switchMultilevelV3.switchMultilevelGet()	// position
    secureSequence (cmds)
}

def secure(hubitat.zwave.Command cmd) {
    // state.sec = 1
    
    if (state.sec) {
        // log.debug "Seq secure - $cmd"
        zwave.securityV1.securityMessageEncapsulation().encapsulate(cmd).format()
        // zwaveSecureEncap(supervisedEncap(cmd).format())
    } 
    else {
        // log.debug "Seq unsecure- $cmd"
        cmd.format()
    }
}

def secureSequence(cmds, delay=1500) {
    //log.debug "SeSeq $commands"
    delayBetween(cmds.collect{ secure(it) }, delay)
}	

private currentDouble(attributeName) {
    if(device.currentValue(attributeName)) {
        return device.currentValue(attributeName).doubleValue()
    }
    else {
        return 0d
    }
}

//chainExtensionSpeed	
private processParam7(cmd) {
    def setValue
    if (cmd.scaledConfigurationValue == 1) {
        setValue = "1 - Slowest"
    }
    if (cmd.scaledConfigurationValue == 2) {
        setValue = "2 - Medium Slow"
    }
    if (cmd.scaledConfigurationValue == 3) {
        setValue = "3 - Medium Fast"
    }
    if (cmd.scaledConfigurationValue == 4) {
        setValue = "4 - Fastest"
    }
    log.info "Configuration Report - Chain Extension Speed: ${setValue}"
}

//rainExtensionSpeed	
private processParam8(cmd) {
    def setValue
    if (cmd.scaledConfigurationValue == 1) {
        setValue = "1 - Slowest"
    }
    if (cmd.scaledConfigurationValue == 2) {
        setValue = "2 - Medium Slow"
    }
    if (cmd.scaledConfigurationValue == 3) {
        setValue = "3 - Medium Fast"
    }
    if (cmd.scaledConfigurationValue == 4) {
        setValue = "4 - Fastest"
    }
    log.info "Configuration Report - Chain Extension Speed (Rain): ${setValue}"
}

//calibration
private processParam12(cmd) {
    def setValue
    if (cmd.scaledConfigurationValue == 1) {
        setValue = "1 - Calibrated"
    }
    if (cmd.scaledConfigurationValue == 2) {
        setValue = "2 - Out of Calibration"
    }
    if (cmd.scaledConfigurationValue == 3) {
        setValue = "3 - Encoder Error"
    }
    log.info "Configuration Report - Calibration: ${setValue}"
}

//gotoPosition
private processParam13(cmd) {
    def setValue
    if (cmd.scaledConfigurationValue == 1) {
        setValue = "1 - Goto Maximum"
    }
    if (cmd.scaledConfigurationValue == 2) {
        setValue = "2 - Goto Previous Position"
    }
    log.info "Configuration Report - Goto Position: ${setValue}"
}

//ventilation
private processParam15(cmd) {
    def setValue 
//    setValue = cmd.scaledConfigurationValue.ToString
	log.info "Configuration Report - Ventilation - Close after ${cmd.scaledConfigurationValue} minutes"
}

def poll() { 
    log.info "$device.displayName Polling...."
    def cmds = []
    cmds <<	zwave.basicV1.basicGet()						// get basic	
    cmds << zwave.switchMultilevelV3.switchMultilevelGet()	// position
    secureSequence (cmds)
}

void debugOff() {
   log.warn("Disabling debug logging")
   device.updateSetting("enableDebug", [value:"false", type:"bool"])
}

void zwaveEvent(hubitat.zwave.commands.supervisionv1.SupervisionGet cmd) {
    hubitat.zwave.Command encapCmd = cmd.encapsulatedCommand(commandClassVersions)
    log.debug "$device.displayName Supervision cmd received"
    if (encapCmd) {
        zwaveEvent(encapCmd)
    }
    sendHubCommand(new hubitat.device.HubAction(zwaveSecureEncap(zwave.supervisionV1.supervisionReport(sessionID: cmd.sessionID, reserved: 0, moreStatusUpdates: false, status: 0xFF, duration: 0).format()), hubitat.device.Protocol.ZWAVE))
}

void zwaveEvent(hubitat.zwave.commands.supervisionv1.SupervisionReport cmd) {
    log.debug "supervision report for session: ${cmd.sessionID}"
    if (!supervisedPackets."${device.id}") { supervisedPackets."${device.id}" = [:] }
    if (supervisedPackets["${device.id}"][cmd.sessionID] != null) { supervisedPackets["${device.id}"].remove(cmd.sessionID) }
    unschedule(supervisionCheck)
}

void supervisionCheck() {
    // re-attempt once
/*    if (!supervisedPackets."${device.id}") { supervisedPackets."${device.id}" = [:] }
    
    supervisedPackets["${device.id}"].each { k, v ->
        log.debug "re-sending supervised session: ${k}"
        sendHubCommand(new hubitat.device.HubAction(zwaveSecureEncap(v), hubitat.device.Protocol.ZWAVE))
        supervisedPackets["${device.id}"].remove(k)
    }
*/
}

Short getSessionId() {
    Short sessId = 1
    if (!sessionIDs["${device.id}"]) {
        sessionIDs["${device.id}"] = sessId
        return sessId
    } else {
        sessId = sessId + sessionIDs["${device.id}"]
        if (sessId > 63) sessId = 1
        sessionIDs["${device.id}"] = sessId
        return sessId
    }
}

hubitat.zwave.Command supervisedEncap(hubitat.zwave.Command cmd) {
    if (getDataValue("S2")?.toInteger() != null) {
        hubitat.zwave.commands.supervisionv1.SupervisionGet supervised = new hubitat.zwave.commands.supervisionv1.SupervisionGet()
        supervised.sessionID = getSessionId()
        log.debug "new supervised packet for session: ${supervised.sessionID}"
        supervised.encapsulate(cmd)
        if (!supervisedPackets."${device.id}") { supervisedPackets."${device.id}" = [:] }
        supervisedPackets["${device.id}"][supervised.sessionID] = supervised.format()
        runIn(5, supervisionCheck)
        return supervised
    } else {
        return cmd
    }
}

void zwaveEvent(hubitat.zwave.commands.switchmultilevelv2.SwitchMultilevelStopLevelChange cmd) {
   logDebug("SwitchMultilevelStopLevelChange: $cmd")
   sendHubCommand(
      new hubitat.device.HubAction(zwave.switchMultilevelV1.switchMultilevelGet().format(),
                                    hubitat.device.Protocol.ZWAVE)
   )
}

def on() {
//	log.trace("on")
    open()
}

def off() {
//	log.trace("off")
    close()
}

def open() {
	log.info("$device.displayName Opening")
	setLevel(255)    
}

def close() {
	log.info("$device.displayName Closing")
	setLevel(0)  
}

def setPosition(level) {
    int newLevel=2.55*level
    log.info("$device.displayName Changing Position to $newLevel")
    setLevel(newLevel)
}

def setLevel(level) {
	log.info "$device.displayName Setting Position to value:${level}"
    def cmds = []
	cmds << zwave.basicV1.basicSet(value: level) 
    cmds <<	zwave.switchMultilevelV3.switchMultilevelGet()    
    secureSequence (cmds)
//    runIn(60, "poll")
}

String startPositionChange(direction){
    Integer upDown = direction == "down" ? 1 : 0
    return secure(zwave.switchMultilevelV1.switchMultilevelStartLevelChange(upDown: upDown, ignoreStartLevel: 1, startLevel: 0))
}

List<String> stopPositionChange(){
    return [
            secure(zwave.switchMultilevelV1.switchMultilevelStopLevelChange())
            ,"delay 200"
            ,secure(zwave.basicV1.basicGet())
    ]
}
