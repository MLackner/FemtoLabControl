# FemtoLabControl
LabVIEW project for instrument control and data acquisition in MG222

## Application Structure
```
Launcher
    |
Dashboard  ---------  <Acquisition Module(s)>
    |             /             |
<Device Modules> /              |
    |              \            |
<Device Drivers>    \    Data Write Module
```

## Module Description
All public APIs do not include standard DQMH requests.

### Launcher
*State Machine* | Launches the Dashboard

***
### Dashboard
*DQMH Module* | Start/Stop Device and Acquisition modules. Could show important data (like the spectrum) and could be used to manually control devices

***
### Acquisition Module(s)
*VI(?)* | Set up the devices for data acquisition. Start and abort data acquisition.

***
### Device Modules
#### Spectrometer
*DQMH Module* | SpectraPro Control

##### Public API

**Set Wavelength** | *arg: wavelength (nm)* | *ret: nothing* | Set the spectrometer to a given wavelength. It is not guaranteed that the spectrometer will really go to this wavelength. The input wavelength is immeadiately broadcasted and passed to the Module Data cluster. *TODO: this is a dummy without functionality*

**Set and Get Wavelength** | *arg: wavelength (nm)* | *ret: wavelength (nm)* | Set the spectrometer to a given wavelength. It is ensured that the spectrometer was set to the correct wavelength which is then returned. The output wavelength is immeadiately broadcasted and passed to the Module Data cluster. *TODO: this is a dummy without functionality*


#### Camera
*DQMH Module* | PI-MAX 4 Control

##### Known Issues

Prior to preparing the file for saving the image from the Module Data cluster via the FileIO module, you must have an image with the correct dimensions in the Module data cluster. Otherwise the file will be prepared with incorrect dimensions for the image and thus the Write to File request will error.

##### Public API

**Get Image** | *arg: nothing* | *ret: 2D Array UInt16* | Get the Image from the camera. The image is broadcasted and put stored in the Module Data cluster. *TODO: this is a dummy without functionality*

***
### FileIO
*DQMH Module* | Write data to disk

Specifiy where data gets written by default in the `config.ini` file under `[FileIO] -> datafolder`. This configuration is loaded on initialization of the module. Make sure this folder exists.

#### Public API

---

**Prepare File** `| arg: data::Variant[, dims::Array{Int,1}, filepath::Path, name::String] | ret: filepath::Path, success::Bool`

Generates an empty HDF5 file with a group/dataset structure resembling the *datacluster* argument. If connected, the datasets will have additional dimensions *dims* in addition to their native dimensions. We can write to this file with the **Write to File** VI. If no file path is specified the file will be saved to the standard path defined in the module configuration. Success is checked by determining if there was an error. The contents of the *.h5 file are not checked against the content of the data cluster.

If *name*, but not *path*, is specified the data gets written to `<datafolder>/<year>-<month>-<day>/<name>/<n>.h5`, where `<n>` is the n'th `.h5` file in the folder.

> ⚠️ **Do not delete any `.h5` files if you intend to write additional files to this folder. The `Prepare File` request counts the `.h5` files in that folder and creates the next file name one based on that.**

If *path* is specified *name* will be ignored and the file will be saved to the specified path. If neither *path* nor *name* is specified the file will be saved at `<datafolder>/<year>-<month>-<day>/noname/<n>.h5`.

The generated path is returned by the request. Wire this output to the `Write to File` VI.

---

**Write to File** `| arg: path::Path, data::Variant[, indices::Array{Int,1}] | ret: success::Bool`

Writes data to the file specified by `path`. Make sure the file was initialized correctly with the `Prepare File` VI.

---
---

## Developer Infos

### Add a new "Get Module Data" Request and Wait for Reply

1. Make the Module Data cluster public. To do that move the "Module Data--cluster.ctl" from "Modules/*Module Name*/Private" to "Modules/*Module Name*/Public API" or any other non-private virtual folder.
2. Create a new "Request and Wait for Reply" for the module in question (Tools -> Delacor -> DQMH -> Event -> Create New DQMH Event...) 
3. Name it "Get Module Data" and insert documentation
4. In the "Reply Payload Window" insert the "Module Data--cluster.ctl" and rename it to the name of the module. The name of the cluster will translate to a group name in the saved HDF5 file, so make sure it's unique. (Therefore rename it to the name of the module)
5. Click "OK"
6. In the "Message Handling Loop", connect the *Module Name* input of the "Bundle By Name" node to the module Data cluster. Also create an error constant for the "Get Module Data_error" input.
7. Set up the event case in the "Test *Module Name* API".