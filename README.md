# GNU Radio runtime + Pothos support

This project contains a very minor fork of the GNU Radio runtime library
to support using GNU Radio blocks within the Pothos framework and GUI.
The gr-runtime fork provides an alternative gnuradio-runtime library
which should be installed on top of an existing GNU Radio installation.

The fork's gnuradio-runtime library will not interfere with stock
GNU Radio usage. This means that the fork is backwards compatible,
and it is possible to merge the fork back into the mainline GNU Radio
(with some minor build system work).

##Dependencies

* A complete GNU Radio install (v3.7.10.1)
* The Pothos framework library

##Documentation

Follow standard CMake project build instructions.
After updating and reinstalling GNU Radio,
this gr-runtime project should also be reinstalled.

## Licensing information

All files including the Pothos support files (gnuradio-runtime/lib/pothos/)
are licensed under the GPLv3 and copyrighted to the FSF.

* See COPYING
