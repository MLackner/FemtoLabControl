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
#### SpectraPro
*DQMH Module* | SpectraPro Control

##### Public API

**Set Wavelength** | *arg: wavelength (nm)* | *ret: nothing* | Set the spectrometer to a given wavelength. It is not guaranteed that the spectrometer will really go to this wavelength. The input wavelength is immeadiately broadcasted and passed to the Module Data cluster. *TODO: this is a dummy without functionality*

**Set and Get Wavelength** | *arg: wavelength (nm)* | *ret: wavelength (nm)* | Set the spectrometer to a given wavelength. It is ensured that the spectrometer was set to the correct wavelength which is then returned. The output wavelength is immeadiately broadcasted and passed to the Module Data cluster. *TODO: this is a dummy without functionality*


#### PICam
*DQMH Module* | PI-MAX 4 Control

##### Public API

**Get Image** | *arg: nothing* | *ret: 2D Array UInt16* | Get the Image from the camera. The image is broadcasted and put stored in the Module Data cluster. *TODO: this is a dummy without functionality*

***
### Data Write Module
*DQMH Module* | Write data to disk

#### Public API

**Save Data** (TODO) | *arg: datacluster::Cluster* | *ret: success::Bool* | Saves the cluster to disk. Folders should be created automatically based on the name of the experiment (should be given in the Acquisition Module) and the data (+other parameters?). Which format?