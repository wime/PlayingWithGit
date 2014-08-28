# SmartSDK REST interface

The SmartSDK REST interface provides a way to communicate with the Test
Controller SDK using the HTTP protocol. It is packaged as a module `smartsdk.node` 
for the [node.js](http://nodejs.org/download/) webserver. The node.js installer 
should match with the `smartsdk.node` module. E.g. when `smartsdk.node` is compiled 
as a 32 bit module node.js should be also 32 bit. Modules for node.js can be loaded 
with [npm](https://npmjs.org/). Npm is installed by the node.js installer.  

## Usage

The interface is a so-called npm package which defines a set of routes and
implementations. It can be included into [Express](http://expressjs.com/)
web application framework for node.

The package has to be loaded in the project to be able to use it. An example of
this from the `package.json` file from the 'Moog GUI Simulation' project:

``` json
{
  "name": "moog-gui-simulation",
  "version": "42",
  "private": true,
  "dependencies": {
    "express": "3.x",
    "socket.io": "x.x",
    "smartsdkrest": "git+ssh://git@github.com:mooginc/smartsdk-rest"
  }
}
```

This tells npm to retrieve the last version of the smartsdkrest package from the
Github repository. You are able to specify specific tags by appending `#ref` to
the Github url, where `#ref` indicates the branch or tag to use.

After loading the package, you can use it from Express as follows:

``` javascript
// Filename: server.js
var express = require('express');
var app     = express();

// Configure the Express server
app.configure(function() {
  app.use(express.bodyParser());
  app.use(express.methodOverride());
  app.use(app.router);
  app.use(express.static(__dirname + '/public'));
});

// Load and configure the SmartSDK REST package.
var smartsdkrest = require('smartsdkrest')('CONTROLLER_IP_ADDRESS', 'LOCAL_IP_ADDRESS');
smartsdkrest.setRoutes(app);

// Start the server.
var server = app.listen(5000);
```

Run the server using the following command:

``` shell
$ node server.js
```

You are then able to access the API calls described below (see API Reference
section below).

### Using real-time data streaming

The interface also allows for streaming real-time data to the client using
Websockets. In your Express server you can use the
[socket.io](http://socket.io/) library to setup the Websocket connection to the
client. An example of this:

``` javascript
// Filename: server.js
//
// Explanation of used variables:
// - server: the running Express server

var io = require('socket.io').listen(server);
io.set ('log level', 1);
io.sockets.on('connection', function(socket) {

  console.log('WebSocket connection established.');

  // Forward the logging to the client.
  smartsdkrest.remoteDevice.on('log', function(log) {
    socket.emit('log', log);
  });

  // Forward the sampler data to the client.
  smartsdkrest.remoteDevice.on('sample', function(samplerName, packet) {
    socket.emit('sampler:' + samplerName, packet);
  });

  // Forward the subscription data to the client.
  smartsdkrest.remoteDevice.on('subscription', function(subscriptionName, packet) {
    socket.emit('subscription:', packet);
  });
  
});
```

### Using the file listing/download/delete endpoints
There currently are five endpoints to get file listings from the controller (`/log`, `/recordings`, `/sequences` and
`/dyingSeconds`, `/config`). When you call these endpoint they will return a list of files from the controller. Where these files
are located on the controller is transparent for the client. They should not need to know where they are.
These directories are subdirectories of the working dir of the node process by default, but can be overriden in
`config/directories.json`.

To download or delete log files use the `/log/:filename` endpoint with a GET or a DELETE request respectively.


## Tests
Integration tests for the rseful interface running against an emulated controller.
### Running the tests:

* Extract a CueingMiddleware emulator in test/CMW.
The executable must be in test/CMW, not in a subdirectory.
* Run tests

``` shell
$ npm test
```

## Code coverage

* Prerequisities

`brew install jscoverage` # OS X

`apt-get install jscoverage` # Ubuntu-like Linux

Win guys: follow [http://siliconforks.com/jscoverage/manual.html]

* Generate coverage output

`npm run-scripts coverage`

* Seeing coverage output

`open coverage.html`

## API Reference

#### GET /controller
Get controller info as object.


### GET /stations
Get all station names of the remote device.


### GET /stations/:stationName
Get station information.


#### GET /stations/:stationName/interlocks
Get interlocks state.


#### GET /stations/:stationName/lock
Get lock information.


#### POST /stations/:stationName/lock
Lock a station.


#### POST /stations/:stationName/unlock
Unlock a station.


#### POST /stations/:stationName/force-unlock
Force unlock of a station.


#### GET /stations/:stationName/pressure
Get pressure state of a station.


#### POST /stations/:stationName/pressure
Set pressure state of a station.


#### POST /stations/:stationName/activate
Activate a station.


#### POST /stations/:stationName/deactivate
Deactivate a station.


#### GET /stations/:stationName/channels
Get all channel names of a station. This channels are used for signal generation.


#### GET /stations/:stationName/manifolds
Get all manifold names of a station.


#### PUT /stations/:stationName/save
Save changed settings on the configuration files on the real-time.


#### GET /stations/:stationName/messages
Get all failure messages of a station.


#### GET /stations/:stationName/property
Get all properties of a station, if no parameters are used. 
Using the parameters allows filtering in the properties:

Parameters:

* `nodeStart` (optional): node in property tree to start searching. Without the query, the search is started
  from the highest level node.
* `levelCount`: depth of search in property tree [(use a value of -1 for a search until the bottom level node.)]
* `purpose` (optional): property purpose (e.g.: TODO)

Example(s):

* Get full property list:  
  `curl -X GET "http://HOST:PORT/stations/STATION_NAME/property"`    

* Get a filtered property tree:
  `curl -X GET "http://HOST:PORT/stations/STATION_NAME/property?nodeStart=System.Model&levelCount=0&purpose=Tuning"`


#### GET /stations/:stationName/property/:propertyPath
Get the value and attributes (as object) of a property specified by a path. Note that the :propertyPath can
also be a comma separated value in which case a value of more properties can be
returned in an array in one go. Only the value of the properties are then returned.

Example(s):

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/property/System.ModeCommand"`
* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/property/System.RunCommand"`
* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/property/
  System.Model.ActualBackwardKinematics.ActuatorMinimumPosition,
  System.Model.ActualBackwardKinematics.ActuatorMaximumPosition"`

  
#### PUT /stations/:stationName/property/:propertyPath
Set value of the property defined by `:propertyPath`. Note that the :propertyPath can
also be a comma separated value in which case an array of values of more properties can be
set in one go. 

You can set a value on a property by posting a json string with key 'value'
to the endpoint url. The json looks like this `{ "value": "42" }`, where `42` is
the value of the property.

This function also has the option to set the attribute (increment, min and max value) 
of the property defined by `:propertyPath`. Only applies when one property is set up.

You can change the increment, maximum value or minimum value a property by posting a json string 
to the endpoint url with key respectively 'increment', 'max' or 'min'.

Example(s):

* Set the emulator to test mode by setting `System.ModeCommand` property to `1`:  
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"value": "1"}' "http://HOST:PORT/stations/STATION_NAME/property/System.ModeCommand"`  

* Tell the system to go into standby mode by setting the `System.RunCommand`
  property to `2`:  
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"value": "2"}' "http://HOST:PORT/stations/STATION_NAME/property/System.RunCommand"`  

* Engage the system by setting the `System.RunCommand` property to `4`:
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"value": "4"}' "http://HOST:PORT/stations/STATION_NAME/property/System.RunCommand"`
  
* Change actuator minimum and maximum position at the same time:
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"value": ["1.1","1.2"]}' 
  "http://HOST:PORT/stations/STATION_NAME/property/
  System.Model.ActualBackwardKinematics.ActuatorMinimumPosition,
  System.Model.ActualBackwardKinematics.ActuatorMaximumPosition"`
  
* Change actuator minimum position to `1.0`, set the increment value to `0.3` and set the maximum value to `2.0`:
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"value": "1.1","increment": "0.3","max": "2.0"}' 
  "http://HOST:PORT/stations/STATION_NAME/property/System.Model.ActualBackwardKinematics.ActuatorMinimumPosition"`


#### GET /stations/:stationName/commonProperty
Retieves the part of the property tree that is of the type UserProperties: the common part of the property tree.


#### PUT /stations/:stationName/runCommand
Change the run command of the top controller. The command needs to be specified by posting a json string with as key 
"command" to the endpoint url. Both a value (number) and the string representation of it can be given as a command.

Example(s):

* Change the run command of controller `System` to 2:  
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"command": "2"}' "http://HOST:PORT/stations/STATION_NAME/runCommand"`  
  
* Change the run command of controller `System` to "Engage":  
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"command": "Engage"}' "http://HOST:PORT/stations/STATION_NAME/runCommand"`  


#### GET /stations/:stationName/runCommand
Get the list giving the path of controllers that has a run command.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/runcommand"`


#### GET /stations/:stationName/runCommand/:runCommandPath
Get the run command properties of the controller with path ":runCommandPath". The properties are returned as 
object with elements: path, value, label and commands (the list which defines the string<->value mapping).


#### GET /stations/:stationName/modeCommand
Get the list giving the path of controllers that has a mode command. By adding the optional query `tree` in the
address, the available mode command paths are returned in tree form.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/modecommand"`
* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/modecommand?tree"`


#### PUT /stations/:stationName/modeCommand/:modeCommandPath
Change the mode command of a controller. The controller path is defined by 
":modeCommandPath". The command needs to be specified by posting a json string with as key 
"command" to the endpoint url. Both a value (number) and the string representation of it 
can be given as a command.

Example:

* Change the mode command of controller `System.Bridge.Proxy` to `0` or `Real`:  
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"command": "0"}' "http://HOST:PORT/stations/STATION_NAME/modeCommand/System.Bridge.Proxy"`
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"command": "Real"}' "http://HOST:PORT/stations/STATION_NAME/modeCommand/System.Bridge.Proxy"`

  
#### GET /stations/:stationName/modeCommand/:modeCommandPath
Get the mode command properties of the controller with path ":modeCommandPath". The properties are returned as 
object with elements: path, value, label and commands (the list which defines the string<->value mapping).


#### GET /stations/:stationName/stateMachine
Get the list giving the path of controllers that has a statemachine.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/statemachine"`


#### GET /stations/:stationName/stateMachine/:stateMachine
Get the state of the controller with path ":stateMachine". The properties are returned as 
object with elements: path, value, label and statemachine (the list which defines the string<->value mapping).


#### POST /stations/:stationName/subscription
Add property subscription.

Parameters:

* `propertyPath`: the properties to be added for subscription.
* `identifier`:  the identifier for the subscription. If a subscription with the identifier already exists, this subscription will 
  be revoked and a new subscription will be made for the prescribed properties.
* `packetFrequency`: the frequency in Hz used for sending packets to the client. A packet frequency of '-1' means that the packet will
  only be sent when a property has changed value.
  
Example:

* `curl -v -H "Content-Type: application/json" -X POST -d '{"propertyPath": ["System.RunCommand","System.ModeCommand"],"identifier":"test"}' "http://HOST:PORT/stations/STATION_NAME/subscription"`
* `curl -v -H "Content-Type: application/json" -X POST -d '{"propertyPath": ["System.RunCommand"],"packetFrequency": ["0.5"],"identifier":"test"}' "http://HOST:PORT/stations/STATION_NAME/subscription"`


#### GET /stations/:stationName/subscription
Get a list of all available subscription names (identifiers). Also the SDK subscription ID will be returned.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/subscription"`


#### GET /stations/:stationName/subscription/:subscriptionName
Get a list of all currently subscribed properties + its property ID that is associated with subscription name (identifier) `:subscriptionName`.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/subscription/SUBSCRIPTION_NAME"`


#### DELETE /stations/:stationName/subscription/:subscriptionName
Revoke the property subscription of all properties associated with subscription name (identifier) `:subscriptionName`.

Example:

* `curl -X DELETE "http://HOST:PORT/stations/STATION_NAME/subscription/SUBSCRIPTION_NAME"`


#### PUT /stations/:stationName/sampler
Setup default instructions for sampler. Instruction for the sampler are specified by posting a json string with as key the instruction 
name to the endpoint url. The available instructions are given below.

Parameters:

* `sampleFrequency`: The frequency in Hz used for collecting samples.
* `packetFrequency`: The frequency in Hz used for sending packets to the client.
* `filter`: A switch (false or true) to tell the controller to filter the signal or not.

Example:

* `curl -v -H "Content-Type: application/json" -X PUT -d '{"sampleFrequency": "10","packetFrequency": "5","filter": "true"}' "http://HOST:PORT/stations/STATION_NAME/sampler"`


#### POST /stations/:stationName/sampler
Start a sampler. The function returns the unique identifier :samplerName. Instruction for the sampler are specified by posting 
a json string with as key the instruction name to the endpoint url. The available instructions are given below.

Parameters:

* `propertyPath`: An array consisting of the properties that are to be sampled. A property name is the complete path to a property starting from *System*. 
  (Optional when a sampler with the same identifier already exists: without this parameter, then it will be assumed that the same properties are to be sampled)
* `sampleFrequency`: The frequency in Hz used for collecting samples. (Optional: without this parameter, the default instructions will be assumed)
* `packetFrequency`: The frequency in Hz used for sending packets to the client. (Optional: without this parameter, the default instructions will be assumed)
* `identifier`: The identifier for the sample window. If a sample window with the identifier already exists, the sampler will be closed and a new sampler will be made for 
  the prescribed properties. Required when you want to change sampleFrequency and/or packetFrequency (Optional: without this parameter, a unique name will be generated 
  to identify the sampler)

Example:

* `curl -v -H "Content-Type: application/json" -X POST -d '{"propertyPath": ["System.outHourCounterSystemActive","System.Model.ActualBackwardKinematics.outPlatformPositionTx"],"identifier": "test"}' "http://HOST:PORT/stations/STATION_NAME/sampler"`


#### GET /stations/:stationName/sampler
Get the names of all running samplers of a station. (only those that are defined in the current session)

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/sampler"`


#### DELETE /stations/:stationName/sampler
Close/stop all running samplers of a station. (does not work correctly yet when multiple users are logged in)

Example:

* `curl -X DELETE "http://HOST:PORT/stations/STATION_NAME/sampler"`


#### GET /stations/:stationName/sampler/:samplerName
Get the names of all properties sampled by the sampler identified by the name :samplerName.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/sampler/SAMPLER_NAME"`


#### DELETE /stations/:stationName/sampler/:samplerName
Close/stop the sampler identified by the name :samplerName.

Example:

* `curl -X DELETE "http://HOST:PORT/stations/STATION_NAME/sampler/SAMPLER_NAME"`


#### DELETE /stations/:stationName/sampler/:samplerName/:propertyPath
Add a single property to the sampler identified by the name :samplerName.

Example:
* `curl -X PUT "http://HOST:PORT/stations/:stationName/sampler/:samplerName/System.Model.ActualBackwardKinematics.outPlatformPositionTx"`

If the parameter 'newPropertyPath' is defined, a existing property will be replaced by a new one.

Example:
* `curl -v -H "Content-Type: application/json" -X PUT -d '{"newPropertyPath": "System.Model.ActualBackwardKinematics.outPlatformPositionTy"}' "http://HOST:PORT/stations/:stationName/sampler/:samplerName/System.Model.ActualBackwardKinematics.outPlatformPositionTx"


#### DELETE /stations/:stationName/sampler/:samplerName/:propertyPath
Remvoe a single property from the sampler identified by the name :samplerName.

Example:

* `curl -X DELETE "http://HOST:PORT/stations/STATION_NAME/sampler/SAMPLER_NAME/System.Model.ActualBackwardKinematics.outPlatformPositionTx"`


#### POST /stations/:stationName/recording
Start a recording. The properties to be recorded are specified by posting a json string with as key "propertyPath"
to the endpoint url. A unique name will be generated to identify the recording. When the recording is finished or stopped
a message is sent via the WebSocket consisting of the recording ID and the file location.
Instruction for the recording are specified by posting a json string with as key the instruction name to the endpoint url. Available 
instructions are: "targetIP", "recordingFolder", "fileName", "sampleFrequency", "duration", "rememberLastSeconds", "resamplingMethod",
"useControllerFreq" and "monitorSet". All instructions are optional.

Parameters:

* `propertyPath`: An array of properties that are to be recorded. A property
  name is the complete path to a property starting from *System*.
* `targetIP`: The IP address of the machine on which the recording file will be stored 
  (when not provided, the recording will be stored on the controller).
* `recordingFolder`: The path of the folder used for storing the recording file.
  (when not provided, the default path is assumed).
* `fileName`: The name used to store the file.
  (when not provided, a filename will be generated based on the current date and time).
* `sampleFrequency`: The frequency in Hz used for collecting samples. 
  Recommended is to use the controller frequency (see function controllerFreq)
* `duration`: Duration of the recording (in seconds).
* `rememberLastSeconds`: The last seconds of the recording that will be kept (only whole numbers is possible).
* `resamplingMethod`: The resampling method (LinearInterpolation or PolyphaseFIR).
* `useControllerFreq`: Use the controller frequency for collecting samples (true or false, sampleFrequency will be disregarded when true).
* `monitorSet`: Record a default set of properties. Available monitor sets can be retrieved using the GET function at the `/monitorSet` endpoint.
  If both "monitorSet" and "propertyPath" are provided as input, both sets will be combined for the recording.

Example:

* `curl -v -H "Content-Type: application/json" -X POST -d '{"propertyPath": ["System.outHourCounterSystemActive","System.Model.ActualBackwardKinematics.outPlatformPositionTx"]}' "http://HOST:PORT/stations/STATION_NAME/recording"`


#### GET /stations/:stationName/recording
Get the names of all running recordings of a station.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/recording"`


#### GET /stations/:stationName/recording/:recName
Get info of a recording. If false is returned the defined recording has stopped running or does not exist.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/recording/RECORDING_NAME"`


#### DELETE /stations/:stationName/recording/:recName
Stop (if still running) the recording identified by the unique name :recName.

Example:

* `curl -X DELETE "http://HOST:PORT/stations/STATION_NAME/recording/RECORDING_NAME"`


#### PUT /stations/:stationName/sequence
Initialize/load a sequence. 

Example:

* `curl -v -H "Content-Type: application/json" -X PUT -d '{"seqName": "sequence.seq"}' "http://HOST:PORT/stations/:stationName/sequence"`


#### GET /stations/:stationName/sequence
Get the status of the sequence player. An object will be returned giving the state as a value and the associated label.


#### POST /stations/:stationName/sequence
Start the loaded sequence. 

Example:

* `curl -X POST "http://HOST:PORT/stations/:stationName/sequence"`


#### DELETE /stations/:stationName/sequence
Stop a sequence. 

Example:

* `curl -X DELETE "http://HOST:PORT/stations/:stationName/sequence"`


#### POST /stations/:stationName/fileReplay
Start a file replay. Instruction for the file replay are specified by posting a json string with as key the instruction name to the endpoint 
url. Available instructions are: "driveFileFolder", "driveFileName", "sequenceFolder", "sequenceName", "recordingFolder", "recordingName",
"driveFileChannelMapping", "recordingProperties", "waitTime", "repeats" and "fileReplayMode". 
File replay makes use of the sequence player so the `/stations/:stationName/sequence` endpoint can be used for stopping the file replay
or retrieving info about the fileReplay.

Parameters:

* `driveFileFolder`: The path of the folder where the drive file is located.
  (optional: when not provided, the default path for recordings is assumed).
* `driveFileName`: The name of the drive file which needs to be replayed.
* `sequenceFolder`: The path of the folder where the generated sequence files are stored.
  The generated sequence files will be played using the sequence player.
  (optional: when not provided, the default path for sequences is assumed).
* `sequenceName`: The name used to store the generated sequence files.
  (optional: when not provided, a filename will be generated based on the current date and time).
 `recordingFolder`: The path of the folder used for storing the recording file.
  (optional: when not provided, the default path for recordings is assumed).
* `recordingName`: The name used to store the recording file.
  (optional: when not provided, a filename will be generated based on the current date and time).
* `driveFileChannelMapping`: An array which defines the file replay mapping. The array includes the drive channels that needs 
  to be replayed and the property on which the it should be replayed on. The following format is expected as input
  [{"driveChannel": driveChannel1, "property": property1}, {"driveChannel": driveChannel2, "property": property2},...]
  (optional: when not provided, fileReplayMode will be used for generating the mapping. If fileReplayMode is also not
  provided, all drive channels is automapped to its own property).
* `waitTime`: The time (in seconds) that need to be waited for between consecutive replays of the drive file.
  (optional: when not provided, it is assumed the waitTime is 0 seconds).
* `repeats`: Number of times replaying of the file is to be repeated.
  (optional: when not provided, it is assumed the file replay is only played once).
* `fileReplayMode`: Automaps drive channels to its connected property as defined by the fileReplayMode. Each
  fileReplayMode consists of an independent mapping set. The fileReplayModes and its mapping set are defined
  in the config file "fileReplay.json". (optional)
  
  
#### GET /fileReplayMode
Retrieves all available modes for fileReplay.


#### GET /stations/:stationName/waveforms
Retrieves the available waveforms for the signal generator. By providing the optional query input: "simple",
a smaller list of the available waveforms is returned. By not providing this query input a list including
all available waveforms is returned.


#### GET /stations/:stationName/signals
Retrieves all signals generated by the signalgenerator. The loaded signals are returned as object. The logic of the object is
as follows: channel->mode->signalProperty. The returned signal list includes active (running) signals and inactive, paused signals.


#### PUT /stations/:stationName/signals
Pauses all signals generated by the signal generator. Pausing has the definition that the signal is stopped, but the info regarding the
signal is still returned using the GET function at the `/signals` endpoint. A paused signal can be restarted using the PUT function at 
the `/signalGenerator/:channel/:mode` endpoint keeping the same signal properties.


#### DELETE /stations/:stationName/signals
Stops and removes all signals generated by the signal generator.


#### GET /stations/:stationName/signalGenerator
Retrieves all available channels of the signal generator. The available channels are returned as object. The logic of the 
object is as follows: controlLoop->channel. Every channel belongs to a control loop.


#### GET /stations/:stationName/signalGenerator/:channel
Retrieves all available modes of a channel.


#### POST /stations/:stationName/signalGenerator/:channel/:mode
Starts/Loads a signal on channel :channel and with :mode. Setup for the signalgenerator are specified by posting a json string 
with as key the property name to the endpoint url. Available properties are: `waveform` (e.g. 'Sine' or "Trapezoid'), and `settings`.
The input for `settings` should be in the same format as that returned by the GET function.

Example:

* `curl -v -H "Content-Type: application/json" -X POST -d '{"waveform": "Sine", "settings": [{"name" : "Period", "value" : 2}, {"name" : "Amplitude", "value" : 0.5}, {"name" : "Offset", "value": 0, {"name" : "FadeTime", "value": 1}]}' "http://HOST:PORT/stations/STATION_NAME/signalGenerator/CHANNEL_NAME/MODE_NAME"`


#### PUT /stations/:stationName/signalGenerator/:channel/:mode
Change the properties of a loaded (active or paused) signal on channel :channel with mode :mode. Changes to the signal are specified by 
posting a json string with as key the property name to the endpoint url. Available properties are: `waveform` (e.g. 'Sine' or "Trapezoid'),
`settings` and `action`. All thse parameters are optional. The input for `settings` should be in the same format as that returned by the GET function.
Without the parameter the previous value or status of the property is retained. 
The parameter `action` has three options for input:
* `play`: Re-start the active signal or play the paused signal.
* `pause`: Pause an active signal. Pausing has the definition that the signal is stopped, but the info regarding the signal is still returned 
  using the GET function at the `/signals` endpoint. The signal can be played again keeping the same properties using the input "action" for
  the `action` parameter.
* undefined, or an empty input: Keep the signal in the current state (active or inactive/paused).

Example:

* Pause the signal:
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"action": "pause"}' "http://HOST:PORT/stations/STATION_NAME/signalGenerator/CHANNEL_NAME/MODE_NAME"`
* Play the paused signal or re-start the signal with the period changed to a value of 2s:
  `curl -v -H "Content-Type: application/json" -X PUT -d '{"settings": [{"name" : "Period": 2}], "action": "play"}' "http://HOST:PORT/stations/STATION_NAME/signalGenerator/CHANNEL_NAME/MODE_NAME"`


#### DELETE /stations/:stationName/signalGenerator/:channel/:mode
Stops and removes the signal on channel :channel with mode :mode.


#### GET /stations/:stationName/signalGenerator/:channel/:mode
Retrieves all properties of the signal on channel :channel with mode :mode.


#### GET /stations/:stationName/manualControl
Retrieves all available channels for manual control. The available channels are returned as object. The logic of the 
object is as follows: controlLoop->channel. Every channel belongs to a control loop. Only the `Position` mode of the channel
is available for manual control.


#### PUT /stations/:stationName/manualControl/:channel
Set a value (setpoint) on channel :channel with mode `Position`. The setpoint is specified by posting a json string with as key
"value" to the endpoint url. 

Example:

* `curl -v -H "Content-Type: application/json" -X PUT -d '{"value": "0.2"}' "http://HOST:PORT/stations/STATION_NAME/manualControl/CHANNEL_NAME"`


#### GET /stations/:stationName/manualControl/:channel
Retrieves the setpoint and the feedback of the signal on channel :channel with mode `Position`. The properties are returned as object
with elements: setpoint and feedback.


#### GET /stations/:stationName/activeCycles
Get a list containing all active cycle channels


#### GET /stations/:stationName/cycle
Get a list containing all available cycle channels


#### PUT /stations/:stationName/cycle/:channel
Setup the cycle defined as :channel

Parameters:

* `channel`: The available channels can be retrieved using the function `GET /stations/:stationName/channels`.
* `mode`: An integer which changes the mode (position = 1, velocity = 2 or acceleration = 3) of the cycle for the specified channel
* `propertyName`: The properties of the cycle which you want to change. This applies on the specified channel and the current mode of the channel. Available properties that can be changed can be found in the function `GET /stations/:stationName/cycle/:channel`

Example:

* `curl -v -H "Content-Type: application/json" -X PUT -d '{"mode": "1"}' "http://HOST:PORT/stations/:stationName/cycle/:channel"`

* `curl -v -H "Content-Type: application/json" -X PUT -d '{"value": ["1.1","1.2"], "propertyName": ["Amplitude","Fade out time"]}' "http://HOST:PORT/stations/:stationName/cycle/DofTz"`

* `curl -v -H "Content-Type: application/json" -X PUT -d '{"value": "1.1", "propertyName": "Amplitude", "mode": "2"}' "http://HOST:PORT/stations/:stationName/cycle/:channel"`


#### POST /stations/:stationName/cycle/:channel
Start the cycle defined as :channel is it is not active.
If the cycle is active, this will pause the cycle.

Example: 

* `curl -X POST "http://HOST:PORT/stations/:stationName/cycle/:channel"`


#### DELETE /stations/:stationName/cycle/:channel
Stop the cycle defined as :channel.

Example: 

* `curl -X DELETE "http://HOST:PORT/stations/:stationName/cycle/:channel"`


#### GET /stations/:stationName/cycle/:channel
Get the properties of the cycle. The properties for the current mode of the channel is returned as object.


#### GET /stations/:stationName/alias
Get all alias definitions as object with element aliasName and aliasPath.

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/alias"`


#### PUT /stations/:stationName/alias/:aliasName
Add an alias with name :aliasName which links to the property defined by the parameter :propertyPath.

Example:

* `curl -v -H "Content-Type: application/json" -X PUT -d '{"propertyPath": "System.Model.ActualBackwardKinematics.PositionHF_RRP_X"}' "http://HOST:PORT/stations/:stationName/alias/:aliasName"`
  

#### DELETE /stations/:stationName/alias/:aliasName  
Remove alias with name :aliasName

Example:

* `curl -X DELETE "http://HOST:PORT/stations/STATION_NAME/alias/ALIAS_NAME"`


#### POST /stations/:stationName/scripting
Load a script. The filename (or filepath if not in the current directory) of the
script file needs to be specified by posting a json string with as key "scriptPath" to the endpoint url.
to the endpoint url. A unique name will be generated to identify the script.

Example:

* `curl -v -H "Content-Type: application/json" -X POST -d '{"scriptPath": "test.txt"}' "http://HOST:PORT/stations/STATION_NAME/scripting"`


#### GET /stations/:stationName/scripting
Get all loaded scripts and their corresponding filepath

Example:

* `curl -X GET "http://HOST:PORT/stations/STATION_NAME/scripting"`


#### PUT /stations/:stationName/scripting/:scriptName?action
(Re)Start/kill a script identified by the unique name :scriptName.
The action performed is specified in the query parameter. There are two possibilities for action: `start` and `kill`.

Example:

* Start/restart a script:  
  `curl -X PUT "http://HOST:PORT/stations/STATION_NAME/scripting/SCRIPT_NAME?action=start"`  

* Kill a script:
  `curl -X PUT "http://HOST:PORT/stations/STATION_NAME/scripting/SCRIPT_NAME?action=kill"`  

  
#### DELETE /stations/:stationName/scripting/:scriptName
Kill script (if still running) and remove from the list the script identified by the unique name :scriptName.

Example:

* `curl -X DELETE "http://HOST:PORT/stations/STATION_NAME/scripting/SCRIPT_NAME"`


### GET /monitorSet
Get available monitor sets. A monitor set consists of a set of default properties for recording/sampling. It is defined by a monitor set 
name that can be used as a shortcut for recording/sampling. During startup the monitor set is loaded from the configuration file "monitorSets.json".


### PUT /monitorSet/:monitorSetName
Modify/add the monitor set defined as `:monitorSetName`. Properties to be linked to the monitor set are specified by posting a json string 
array with as key `propertyPath` to the endpoint url.


### DELETE /monitorSet/:monitorSetName
Delete the monitor set defined as `:monitorSetName`.


#### PUT /save/monitorSet
Save the monitor sets in the configuration file "monitorSets.json". The same monitor sets will then be reloaded in later sessions.


#### POST /stream
Load a streamfile. The filename (or filepath if not in the current directory) of the
stream needs to be specified by posting a json string with as key "filePathPath" to the endpoint url.
to the endpoint url. A unique name will be generated to identify the stream.

Example:

* `curl -v -H "Content-Type: application/json" -X POST -d '{"filePath": "srec.fstrm"}' "http://HOST:PORT/stream"`


#### GET /stream
Get all loaded streamfiles and their corresponding filepath

Example:

* `curl -X GET "http://HOST:PORT/stream"`


#### PUT /stream/:streamName?action&outFileName&outFolder&outFormat
Move/copy/convert the stream that is identified by the unique name :streamName.
The action performed is specified in the query parameter.

Parameters:

* `action`: Defines the action to be performed. There are four options: 'move', 'copy', 'convertFile' and 'convertFolder'.
  With 'move' the streamfile is moved to the location specified by the query parameter 'outFileName'.
  With 'copy' the streamfile is copied to the location specified by the query parameter 'outFileName'.
  With 'convertFile' the streamfile is copied to the location specified by the query parameter 'outFileName'. A change in formats will be taken account with for the data.
  With 'convertFolder' all streamfiles within the folder is converted to the format specified by the query parameter 'outFormat'. These copies will be placed in the location specified by the query parameter 'outFolder'

Example:
* Move streamfile:  
  `curl -X PUT "http://HOST:PORT/stream/STREAM_NAME?action=move&outFileName=srec2.fstrm"`  

* Copy streamfile:  
  `curl -X PUT "http://HOST:PORT/stream/STREAM_NAME?action=copy&outFileName=srec2.fstrm"`  
  
* Convert streamfile:  
  `curl -X PUT "http://HOST:PORT/stream/STREAM_NAME?action=convertFile&outFileName=rec.sam"`  

* Convert all streamfiles within a folder:  
  `curl -X PUT "http://HOST:PORT/stream/STREAM_NAME?action=convertFolder&outFolder=file://10.30.156.50/C:/Projects/Applications&outFormat=txt"`  
  
  
#### GET /stream/:streamName?type&toStream&tolerance
Get info regarding the stream that is identified by the unique name :streamName or compare the stream to another stream.
The action performed is specified in the query parameter 'type'.

Parameters:

* `type`: Defines the action to be performed. There are two options: 'info' and 'compare'.
  With 'info' information regarding the streamfile is returned.
  With 'compare' the streamfile is compared to the streamfile identified by the unique name specified by the query parameter 'toStream'. The tolerance for comparison is specified by the query paramter 'tolerance'.

Example:
* Get info regarding streamfile:  
  `curl -X GET "http://HOST:PORT/stream/STREAM_NAME?type=info"`  

* Compare streamfiles:  
  `curl -X GET "http://HOST:PORT/stream/STREAM_NAME?type=compare&toStream=STREAM_NAME2&tolerance=0.001"`  
  

#### DELETE /stream/:streamName?action
Delete streamfile or remove from the list the streamfile that identified by the unique name :streamName.
The action performed is specified in the query parameter 'action'.

Parameters:

* `action`: Defines the action to be performed. There are two options: 'delete' and 'removeID'.
  With 'delete' the streamfile is actually deleted.
  With 'removeID' the streamfile is only removed from the list. Nothing is done to the actual file.

Example:
* Delete streamfile:  
  `curl -X DELETE "http://HOST:PORT/stream/STREAM_NAME?action=delete"`  

* Delete streamID:
  `curl -X DELETE "http://HOST:PORT/stream/STREAM_NAME?action=removeID"`  

