# PyUSB-VirtSpec - WP Spectrometer Emulator

<img width="743" alt="image" src="https://github.com/WasatchPhotonics/pyusb-virtSpec/assets/124081765/93182c73-d7ac-43ae-a2c8-0e69822e4a5e">

## Introduction

This module substitutes PyUSB and provides a virtual spectrometer.

## Background

See the Readme for [PyUSB v1.2.1](https://github.com/WasatchPhotonics/pyusb-virtSpec/blob/6df282104b8acea44539cec690261c5127f76e45/README.rst).

## Usage

### Enlighten

There is a shortcut for using pyusb-virtspec with Enlighten. Simply call the following from your shell.

```
scripts\bootstrap.bat activate virtspec
```

This can simplify testing of UI and plugins. Do not distribute Enlighten with virtspec. Always test with a real spectrometer before releasing.

### Local

Have pyusb-virtSpec in the same folder as Enlighten, Wasatch.PY, or any other Python software.

Replace 'export PYTHONPATH...' with the correct way to set environment variables in your OS.

Optional: Create/activate a conda environment first.

```
pip uninstall pyusb
export PYTHONPATH=../pyusb-virtSpec;$PYTHONPATH
```

That's it. Now you can run Wasatch.PY (simple-demo.py) or a different program and it will talk to a single virtual spectrometer by default.

### Installed 

Optional: Create/activate a conda environment first.

```
pip uninstall pyusb
cd pyusb-virtSpec
python setup.py install
```

That's it. Now you can run ENLIGHTEN or a different program and it will talk to a single virtual spectrometer by default.

## How it works / Customization

It is possible to do several customizations, such as connecting multiple devices at once, emulating specific product/vendor id's, simulating raman, supplying custom eeprom values, and responding to endpoints. I'll start with an explanation of control-flow.

Firstly, upon import the following message is displayed.

```
***** USING VIRTUAL PYUSB BACKEND *****
```

This shows the local environment is configured to use this module instead of the original PyUSB.

Next, `usb.core.find` has been modified to return only the contents of the list `CONNECTED_DEVICES`. This variable is where all virtual devices are specified--it is defined elsewhere, making it rarely necessary to modify `core.py` in the future. One reason to modify `core.py` would be to enable a mixture of real and virtual devices. Currently only virtual devices will be returned by `find`.

### Specifying Devices

The main place for configuration and extension is `usb/wasatchConfig.py`. This code is kind of messy, but the gist of it is as follows. There is a class called `VirtualDevice` which acts as a stand in for a libusb0 backend Device. It inherits from a smaller helper class called `Ignore` because we tend to want to ignore extra messages that are meant for device state configuration. The list `CONNECTED_DEVICES` should be populated with one instance of VirtualDevice per virtual spectrometer.

### Device Meta-Data

Data such as `.serial_number` and `.product` can be modified directly. VirtualDevice handles the bare minimum of control messages in order to pass Enlighten/Wasatch.PY initialization. Specifically, that consists of two `DEVICE_TO_HOST` messages: (1) Request FPGA Compilation Options and (2) Request EEPROM. 

FPGA Options are explained in WasatchPhotonics/Wasatch.PY:wasatch/FPGAOptions.py:38. By default VirtualDevice responds with all ones. 

The EEPROM can be specified by modifying the default parameters of `__init__` of `EEPROM` in `eeprom_gen.py`. This file is a self-contained copy of EEPROM.py from WasatchPhotonics/Wasatch.PY. It is used only for its initializer, `.generate_write_buffers()`, and the list `.write_buffers`.

~For simplicity, wavecal is left blank, so you must use `X-AXIS=pixel` within Enlighten to see a sensible spectra.~

Wavecal is now set to map [0,1024) pixels to [800, 1000) wavelengths. Linear mappings such as this are trivial to generate manually: `[start_wavelength, (end_wavelength-start_wavelength)/1024, 0, 0]`

### Supporting Additional Control Messages

Currently most control messages are ignored. For example `ACQUIRE` is ignored, since the virtual spec can always reply to read with simulated spectra. It doesn't need time to integrate or to do anything state-wise. It may be desireable to do more than ignore control messages in some cases. For example implementing `SET_INTEGRATION` and `GET_INTEGRATION` would be trivial to do in VirtualDevice, and it would allow for testing device-level parameter persistence in Enlighten.

Other control messages can be implemented by copying `unhandled ctrl_t` messages as a starting point. They are formatted similarly to how they would be as conditionals. Be sure to remove `var == None` from those conditionals in order to properly capture the requests. Integer returns types are usually specified using two item lists: [LSB, MSB]. Check the specification or drivers to get more info about what exactly will be expected for a given new endpoint.

### Customizing Spectra

Customizing spectra is very simple and minimal. You can either modify variables `peaks` and `noise` in wasatchConfig. Or you can fill `pixels` with arbitrary integer-only data. Take care that `len(pixels)` corresponds correctly with the device PID that you specified. By default the PID is `0x1000` and `len(pixels)` should be 1024.

There is a known problem when setting peak heights in the ballpark of 40k-60k counts. For some reason the simulated spectra gains visible jagged artifacts at this scale. The exact same generation function scaled down to ~2-5k counts removed the artifacts for me.

# Testing

This section is brief guide about using pyusb-virtSpec in testing. The motivation of this module is to reduce spectrometer dependence for high-level tests that don't depend on any meaningful spectra. For instance saving and loading measurements, running plugins, and confirming the graph still shows. This virtual spectrometer enables parts of Enlighten that are normally unavailable when no spec is connected.

Characterizing real spectrometer outcomes using virtual test results should only be attempted with a strong understanding of this software and WP drivers, but sometimes it's obvious. If the virtual spec suddenly breaks after a code change, you've must likely done something to break compatibility with a real spec as well.
