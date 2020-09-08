# FemtoLabControl
LabVIEW project for instrument control and data acquisition in MG222

## Application Structure

The application is build up from different modules. The **Dashboard** module is used to control the setup as a whole. Devices can be read out and controlled by the user from this module. Devices are working independently from one another and are grouped under **Device Modules** in the list below. All devices work independently from one another which increases maintainability of the application.
The device modules expose an API that can be used in the whole scope of the project. For example the **Spectrometer Module** exposes a VI called `Set Wavelength` which allows you to set the wavelength of the spectrometer. This becomes relevant if additional modules are added (for example an acquisition VI) that communicate with other modules.

Another modules are the **Notification Module**, which is responsible to send notifications to the user. Currently this is limited to receiving messages via the <a href="https://play.google.com/store/apps/details?id=org.telegram.messenger&hl=en">Telegram</a> messenger.

The **FileIO** module is responsible for saving acquired data.

    - Dashboard
        - \<Acquisition Modules\>
            - Acquisition Example
        - \<Device Modules\>
            - Camera
            - Spectrometer
            - ESP301
        - Notification Module
        - FileIO

Several of these are <a href="https://delacor.com/products/dqmh/">DQMH Modules</a>. The benefit of using this kind of framework is that modules can work independently of one another, while also being able to intercommunicate. For example one does not necessarily need the delay stages to be connected in order to use the Dashboard. Also, because this is a standardized framework in terms of 'how things work' it is relatively easy to understand the LabVIEW "code" once you get the hang of it, although - admittedly - the learning curve is quite high. There are however some well done tutorials available.

## Setup

> ℹ️ | This section only applies if you pull a fresh copy of the GitHub repository to your hard drive.

### Picam Driver

The driver can be downloaded from the <a href="ftp://ftp.princetoninstruments.com/Public/Software/Official/PICam">Princeton Instruments Website</a>. After installation of the driver navigate to `C:\Program Files\Common Files\Princeton Instruments\Picam\Runtime`, make a **copy** of `Picam.dll` and rename it to `Picam64.dll`. This is because the LabVIEW wrapper for the Picam driver has to automatically chooses between the 32 and 64bit version, depending on which LabVIEW version is running and `Picam.dll` is not recognized as the 64bit version of the `.dll`.

### ESP301 Driver

Download the driver <a href="https://www.newport.com/medias/sys_master/images/images/h24/hfe/9044102840350/ESP301-GUI-V2.0.0.3.zip">here</a>.

### Notification Module

For the Notification module to work correctly you need to create a `secret_config.ini` file in the FemtoLabControl parent folder (where the `config.ini` is). 

> ⚠️ **Do not put this file under version control or make it public otherwise!**

Content should be:
```ini
[Notification]
;make sure you prepend 'bot' to your token (considered bug?)
token=bot<your-telegram-bot-token>
password=<your-password-for-joining-groups>

[ntf]
; notification group for non-critical messages

[ntfcrit]
; notification group for critical messages
```

## Module Description
All public APIs do not include standard DQMH requests.

### Launcher
*State Machine* | Launches the Dashboard and the FileIO module.

***
### Dashboard
*DQMH Module* | Start/Stop Device and Acquisition modules. Could show important data (like the spectrum) and could be used to manually control devices

### Notification
*DQMH Module* | Send and receive notifications to and from the outside world

There are the following notification groups:
* `ntf`: for non-cirtical messages like acquisition status etc.
* `ntfcrit`: for critical messages like malfunctions.

You can subscribe to the groups by sending messages to your Telegram bot:
>`/sub <group> <password>`

Unsubscribe via:
>`/unsub <group>`


***
### Acquisition Module(s)

#### Acquisition Example
*State Machine* | Scan the delay stage.

***
### Device Modules

All device modules (should) have a `Get Module Data` request. This returns the data that is stored in the specific module's internal data cluster.


#### Spectrometer
*DQMH Module* | SpectraPro Control

##### Public API

`Set Wavelength` **| arg: wavelength (nm) | ret: wavelength (nm) |** Set the spectrometer to a given wavelength. It is not guaranteed that the spectrometer will really go to this wavelength. The set wavelength in nm is returned and broadcasted.

`Set Slit Width` **| arg: slit width (um) | ret: slit width (um) |** Set the spectrometer to a given slit width. It is not guaranteed that the spectrometer will really go to this slit width. The set slit width in um is returned and broadcasted.

`Set Grating` **| arg: grating id | ret: grating id |** Set the spectrometer to a given grating id. It is not guaranteed that the spectrometer will really go to this grating id. The set grating id is returned and broadcasted.

****

#### Camera
*DQMH Module* | PI-MAX 4 Control

##### Known Issues

Prior to preparing the file for saving the image from the Module Data cluster via the FileIO module, you must have an image with the correct dimensions in the Module data cluster. Otherwise the file will be prepared with incorrect dimensions for the image and thus the Write to File request will error.

##### Public API

`SetGet <Value Type> Value` **| args: set/get::Enum, broadcast::Bool, parameter::PicamParameter, [value::<Value Type>] |** Determine if the value is set or read. If the value is set, it is read as well to ensure it is set correctly. Select if it broadcasted or not via the `Broadcast` argument. In case `Set` is chosen, the module will broadcast a `Status Updated` message. 

> ℹ Refer to the [Picam Programming Manual](https://www.princetoninstruments.com/wp-content/uploads/2020/04/PICAM-5.x-Programmers-Manual-Issue-8-4411-0161-2.pdf#page=295) for the correct value type of the specified parameter. For example `IntensifierGain` is an *Integer* parameter. So one has to use the `SetGet Integer Value` request.

`Acquire Image` **|** Acquires an image with the internal camera settings. This operation is asynchronous and will error if the time needed for accumulating the data exceeds the timeout of the Camera module.

> The timeout value can be changed from the LabVIEW project under `My Computer/Modules/Camera.lvlib/Private/Module Timeout--constant.vi`

`Get Image` **| ret: image::Array{UInt16,2d} |** Get the Image from the camera. The image is broadcasted and put stored in the Module Data cluster.

`Set ROI` **| arg: roiarray::Array{PicamRoi} | ret: roiarray |** Set the ROIs. For valid ROIs refer to the [PIMAX4 System Manual](https://www.princetoninstruments.com/wp-content/uploads/2020/04/PI-MAX4-System-Manual-Issue-10-4411-0139.pdf#page=70).

***
#### FileIO
*DQMH Module* | Write data to disk

Specifiy where data gets written by default in the `config.ini` file under `[FileIO] -> datafolder`. This configuration is loaded on initialization of the module. Make sure this folder exists.

##### Public API

`Prepare File` **| arg: data::Variant[, dims::Array{Int,1}, filepath::Path, name::String] | ret: filepath::Path, success::Bool**

Generates an empty HDF5 file with a group/dataset structure resembling the *datacluster* argument. If connected, the datasets will have additional dimensions *dims* in addition to their native dimensions. We can write to this file with the **Write to File** VI. If no file path is specified the file will be saved to the standard path defined in the module configuration. Success is checked by determining if there was an error. The contents of the *.h5 file are not checked against the content of the data cluster.

If *name*, but not *path*, is specified the data gets written to `<datafolder>/<year>-<month>-<day>/<name>/<n>.h5`, where `<n>` is the n'th `.h5` file in the folder.

> ⚠️ **Do not delete any `.h5` files if you intend to write additional files to this folder. The `Prepare File` request counts the `.h5` files in that folder and creates the next file name one based on that.**

If *path* is specified *name* will be ignored and the file will be saved to the specified path. If neither *path* nor *name* is specified the file will be saved at `<datafolder>/<year>-<month>-<day>/noname/<n>.h5`.

The generated path is returned by the request. Wire this output to the `Write to File` VI.


`Write to File` **| arg: path::Path, data::Variant[, indices::Array{Int,1}] | ret: success::Bool**

Writes data to the file specified by `path`. Make sure the file was initialized correctly with the `Prepare File` VI.

---
---

## Developer Infos

### DQMH Tutorials

* [Tutorial series on YouTube on DQMH](https://www.youtube.com/watch?v=TAhixRaj2Gg&list=PLmF-6jvwRvVOpNdPcRzskvnGbW79ug6AZ)
* [DQMH best practices](https://delacor.com/dqmh-documentation/dqmh-best-practices/)
* [DQMH dos and don'ts](https://delacor.com/simple-dqmh-dos-and-donts/)
* [DQMH helper loops](https://www.hampel-soft.com/blog/dqmh-actors-self-messaging-or-helper-loops/)

### Add a new "Get Module Data" Request and Wait for Reply

1. Make the Module Data cluster public. To do that move the "Module Data--cluster.ctl" from "Modules/*Module Name*/Private" to "Modules/*Module Name*/Public API" or any other non-private virtual folder.
2. Create a new "Request and Wait for Reply" for the module in question (Tools -> Delacor -> DQMH -> Event -> Create New DQMH Event...) 
3. Name it "Get Module Data" and insert documentation
4. In the "Reply Payload Window" insert the "Module Data--cluster.ctl" and rename it to the name of the module. The name of the cluster will translate to a group name in the saved HDF5 file, so make sure it's unique. (Therefore rename it to the name of the module)
5. Click "OK"
6. In the "Message Handling Loop", connect the *Module Name* input of the "Bundle By Name" node to the module Data cluster. Also create an error constant for the "Get Module Data_error" input.
7. Set up the event case in the "Test *Module Name* API".