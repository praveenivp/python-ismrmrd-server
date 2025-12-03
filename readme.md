##  <a name='TableofContents'></a> Table of Contents
<!-- vscode-markdown-toc -->
* 1. [Getting Started](#GettingStarted)
	* 1.1. [Reconstruct a phantom raw data set using the MRD client/server pair](#ReconstructaphantomrawdatasetusingtheMRDclientserverpair)
	* 1.2. [Creating a custom reconstruction/analysis module](#Creatingacustomreconstructionanalysismodule)
		* 1.2.1. [Adding a raw k-space filter](#Addingarawk-spacefilter)
		* 1.2.2. [Adding an image processing filter](#Addinganimageprocessingfilter)
	* 1.3. [Using raw data from an MRI scanner](#UsingrawdatafromanMRIscanner)
	* 1.4. [Using DICOM images as input data](#UsingDICOMimagesasinputdata)
* 2. [Code design](#Codedesign)
* 3. [Saving incoming data](#Savingincomingdata)


<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->
##  1. <a name='GettingStarted'></a>Getting Started
###  1.1. <a name='ReconstructaphantomrawdatasetusingtheMRDclientserverpair'></a>Reconstruct a phantom raw data set using the MRD client/server pair

1. Set up a working environment with venv
    ```
    git clone 
    cd python-ismrmrd-server
    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements
    ```
1. In a command prompt, generate a sample raw dataset:
    ```
    python generate_cartesian_shepp_logan_dataset.py -o phantom_raw.h5
    ```

    MRD data is stored in the HDF file format in a hierarchical structure in groups.  The above example creates a ``dataset`` group containing:
    ```
    /dataset/data   Raw k-space data
    /dataset/xml    MRD header
    ```

1. Start the server in verbose mode by running:
    ```
    python main.py -v -p 9010
    ```

1. Start the client and send data to the server for reconstruction:
    ```
    python client.py -G dataset -p 9010 -o phantom_img.h5 phantom_raw.h5
    ```
    The ``-G`` argument specifies the group name in the output file, the ``-o`` argument specifies the output file, and the last argument is the input file.

    MRD image data are also stored in HDF files arranged by groups. If the ``-G`` argument is not provided, a group name will be automatically created with the current date and time.  This may be useful when running the client multiple times, as multiple groups, organized by date/time, can be stored in the same output file.  Images are further grouped by series index, with a sub-group named ``image_x``, where x is image_series_index in the ImageHeader.  For example:
    ```
    /dataset/image_0/data         Image data
    /dataset/image_0/header       MRD ImageHeader structure
    /dataset/image_0/attributes   MRD MetaAttributes text
    ```

1. The [mrd2gif.py](mrd2gif.py) program can be used to convert an MRD Image file into an animated GIF for quick previewing:
    ```
    python mrd2gif.py phantom_img.h5
    ```
    A GIF file (animated if multiple images present) is generated in the same folder as the MRD file with the same base file name and the group and sub-groups appended.

    The reconstructed images in /tmp/phantom_raw.h5 can be opened in any HDF viewer such as https://www.hdfgroup.org/downloads/hdfview/.  In Python, the [ismrmrd-python-tools](https://github.com/ismrmrd/ismrmrd-python-tools) repository has an interactive ``imageviewer`` tool that can be used to view MRD formatted HDF files.  The syntax is:
    ```
    python imageviewer.py phantom_img.h5
    ```

    In MATLAB, the [ISMRMRD](https://github.com/ismrmrd/ismrmrd) library contains helper classes to load and view MRD files.  The files can also be read using MATLAB’s built-in HDF functions:
    ```
    img = h5read('/tmp/phantom_img.h5', '/dataset/image_0/data');
    figure, imagesc(img), axis image, colormap(gray)
    ```

###  1.2. <a name='Creatingacustomreconstructionanalysismodule'></a>Creating a custom reconstruction/analysis module
The MRD server has a modular design to allow for easy integration of custom reconstruction or image analysis code.  The config that is passed by the client (e.g. the `--config` (`-c`) argument in [client.py](client.py)) is interpreted by the server as the "module" that should be executed to parse the incoming data.  For example, a config of `invertcontrast` will select [invertcontrast.py](invertcontrast.py) as the module to be run.  Additional modules can be added simply be creating the appropriately named .py file in the Python path (e.g. the current folder).

If a file/module corresponding to the selecting config cannot be found, the server will fall back to a default config.  The default config can be provided to [main.py](main.py) using the `--defaultConfig` (`-d`) argument.  It is recommended that the default config argument be set in the `CMD` line of the Dockerfile when building an image to indicate the intended config to be run.

####  1.2.1. <a name='Addingarawk-spacefilter'></a>Adding a raw k-space filter
In this example, a Hanning filter is applied to raw k-space data.

1. Create a copy of [invertcontrast.py](invertcontrast.py) named ``filterkspace.py``.  For other workflows, it may be preferable to start with another example such as [simplefft.py](simplefft.py).

1. The NumPy library contains a [Hanning filter](https://numpy.org/doc/stable/reference/generated/numpy.hanning.html) that can be applied to k-space data before the Fourier transform.

1. In the ``process_raw()`` function the ``filterkspace.py`` file, find the [section where raw k-space data is already sorted into a Cartesian grid, just prior to Fourier transform](https://github.com/kspaceKelvin/python-ismrmrd-server/blob/27454bd9f1a2c7fd3928bfa0767840b0d015d988/invertcontrast.py#L177).  Add a Hanning filter and perform element-wise multiplication of the k-space data with the filter:

    **Old code:**
    ```
    # Fourier Transform
    data = fft.fftshift( data, axes=(1, 2))
    data = fft.ifft2(    data, axes=(1, 2))
    data = fft.ifftshift(data, axes=(1, 2))
    ```

    **New code:**
    ```
    # Apply Hanning filter
    logging.info("Applying Hanning filter to k-space data")
    filt = np.sqrt(np.outer(np.hanning(data.shape[1]), np.hanning(data.shape[2])))
    filt = np.expand_dims(filt, axis=(0,3))
    data = np.multiply(data,filt)
    np.save(debugFolder + "/" + "rawFilt.npy", data)

    # Fourier Transform
    data = fft.fftshift( data, axes=(1, 2))
    data = fft.ifft2(    data, axes=(1, 2))
    data = fft.ifftshift(data, axes=(1, 2))
    ```

1. The module used by the server is specified by the ``config`` option on the client side.  The Server class used in this code [attempts to find](https://github.com/kspaceKelvin/python-ismrmrd-server/blob/6684b4d17c0591e64b34bc06fdd06d78a2d8c659/server.py#L105) a Python module matching the name of the config file if it doesn't match one of the default examples.  [Start the server](https://github.com/kspaceKelvin/python-ismrmrd-server#ReconstructaphantomrawdatasetusingtheMRDclientserverpair) and in a separate window, run the client with the ``-c filterkspace`` option to specify the new config:
    ```
    python client.py -c filterkspace -o phantom_img.h5 phantom_raw.h5
    ```

1. Create a GIF preview of the filtered image:
    ```
    python mrd2gif.py phantom_img.h5
    ```

####  1.2.2. <a name='Addinganimageprocessingfilter'></a>Adding an image processing filter
In this example, a high-pass filter is applied to images.

1. Create a copy of [invertcontrast.py](invertcontrast.py) named ``filterimage.py``.  For other workflows, it may be preferable to start with another example such as [analyzeflow.py](analyzeflow.py).

1. The Python [Pillow](https://github.com/python-pillow/Pillow/) library contains a high-pass filter named [FIND_EDGES](https://pythontic.com/image-processing/pillow/edge-detection).  Add the following line to the top of the newly created ``filterimage.py`` to import this library:
    ```
    from PIL import Image, ImageFilter
    ```

1. In the ``process_image()`` function the ``filterimage.py`` file, find the sections where the incoming images are being [normalized](https://github.com/kspaceKelvin/python-ismrmrd-server/blob/6684b4d17c0591e64b34bc06fdd06d78a2d8c659/invertcontrast.py#L261) and [filtered](https://github.com/kspaceKelvin/python-ismrmrd-server/blob/6684b4d17c0591e64b34bc06fdd06d78a2d8c659/invertcontrast.py#L267). The Pillow image filters require images with values in the range 0-255, so replace these two sections as follows:

    **Old code:**
    ```
    # Normalize and convert to int16
    data = data.astype(np.float64)
    data *= maxVal/data.max()
    data = np.around(data)
    data = data.astype(np.int16)

    # Invert image contrast
    data = maxVal-data
    data = np.abs(data)
    data = data.astype(np.int16)
    np.save(debugFolder + "/" + "imgInverted.npy", data)
    ```

    **New code:**
    ```
    # Normalize to range 0-255
    data = data.astype(np.float64)
    data *= 255/data.max()

    # Apply a 2D high-pass filter for each image
    logging.info("Applying high-pass image filter")
    from PIL import Image, ImageFilter
    for iImg in range(data.shape[-1]):
        im = Image.fromarray(np.squeeze(data[...,iImg])).convert('RGB')
        im = im.filter(ImageFilter.FIND_EDGES)
        data[:,:,0,0,iImg] = np.asarray(im)[...,0]

    # Rescale back to 16-bit
    data = data * maxVal/data.max()
    data = data.astype(np.int16)
    np.save(debugFolder + "/" + "imgFiltered.npy", data)
    ```

1. The module used by the server is specified by the ``config`` option on the client side.  The Server class used in this code [attempts to find](https://github.com/kspaceKelvin/python-ismrmrd-server/blob/6684b4d17c0591e64b34bc06fdd06d78a2d8c659/server.py#L105) a Python module matching the name of the config file if it doesn't match one of the default examples.  [Start the server](https://github.com/kspaceKelvin/python-ismrmrd-server#ReconstructaphantomrawdatasetusingtheMRDclientserverpair) and in a separate window, run the client with the ``-c filterimage`` option to specify the new config:
    ```
    python client.py -c filterimage -o phantom_img.h5 phantom_raw.h5
    ```

1. Create a GIF preview of the filtered image:
    ```
    python mrd2gif.py phantom_img.h5
    ```

###  1.3. <a name='UsingrawdatafromanMRIscanner'></a>Using raw data from an MRI scanner
Raw data from MRI scanners can be converted into MRD format using publicly available conversion tools such as [siemens_to_ismrmrd](https://github.com/ismrmrd/siemens_to_ismrmrd), [ge_to_ismrmrd](https://github.com/ismrmrd/ge_to_ismrmrd), [philips_to_ismrmrd](https://github.com/ismrmrd/philips_to_ismrmrd), and [bruker_to_ismrmrd](https://github.com/ismrmrd/bruker_to_ismrmrd).  These can be used as input data for the client as part of streaming MRD framework.

For Siemens data, raw data in .dat format can be converted and processed as follows:
1. Convert a raw data file named ``gre.dat`` into MRD format:
    ```
    siemens_to_ismrmrd -Z -f gre.dat -o gre_raw.h5
    ```

    The following options are used:
    ```
    -Z  Convert all acquisitions from a multi-measurement (multi-RAID) file
    -f  Input file (Siemens raw data .dat file)
    -o  Output file (MRD raw data .h5 file)
    ```

    If the input file is a multi-RAID file, then several output files are created such as ``gre_raw_1.h5`` and ``gre_raw_2.h5``.  The first measurements are dependency data while the main acquisition is in the last numbered file.

1. [Start the server](https://github.com/kspaceKelvin/python-ismrmrd-server#ReconstructaphantomrawdatasetusingtheMRDclientserverpair) and in a separate window, run the client using the converted file:
    ```
    python client.py -c invertcontrast -o gre_img.h5 gre_raw_2.h5
    ```

    Note that the invertcontrast example module only does basic Fourier transform reconstruction and does not support undersampling or more complex acquisitions.

###  1.4. <a name='UsingDICOMimagesasinputdata'></a>Using DICOM images as input data
For image processing workflows, DICOM images can be used as input by converting them into MRD format.  The [dicom2mrd.py](dicom2mrd.py) script can be used to convert DICOMs to MRD, while [mrd2dicom.py](mrd2dicom.py) can be used to perform the inverse.

1. Create a folder containing DICOM files with file extensions .ima or .dcm.  Files can also be organized in sub-folders if desired.

1. Run the dicom2mrd conversion script:
    ```
    python dicom2mrd.py -o dicom_img.h5 dicoms
    ```
    Where the DICOM files are in a folder called ``dicoms`` and an output file ``dicom_img.h5`` is created containing the MRD formatted images.

1. [Start the server](https://github.com/kspaceKelvin/python-ismrmrd-server#ReconstructaphantomrawdatasetusingtheMRDclientserverpair) and in a separate window, run the client using the converted file:
    ```
    python client.py -c invertcontrast -o dicom_img_inverted.h5 dicom_img.h5
    ```

1. Convert the output MRD file back to a folder of DICOMs:
    ```
    python mrd2dicom.py dicom_img_inverted.h5
    ```
##  2. <a name='Codedesign'></a>Code design
This code is designed to provide a reference implementation of an MRD client/server pair.  It is modular and can be easily extended to include additional reconstruction/analysis programs.

- [main.py](main.py):  This is the main program, parsing input arguments and starting a "Server" class instance.

- [server.py](server.py):  The "Server" class determines which reconstruction algorithm (e.g. "simplefft" or "invertcontrast") is used for the incoming data, based on the requested config information.

- [connection.py](connection.py): The "Connection" class handles network communications to/from the client, parsing streaming messages of different types, as detailed in the [MRD documentation](https://ismrmrd.readthedocs.io/en/latest/mrd_messages.html).  The connection class also saves incoming data to MRD files if this option is selected.

- [constants.py](constants.py): This file contains constants that define the message types of the MRD streaming data format.

- [mrdhelper.py](mrdhelper.py): This class contains helper functions for commonly used MRD tasks such as copying header information from raw data to image data and working with image metadata.

- [client.py](client.py): This script can be used to function as the client for an MRD streaming session, sending data from a file to a server and saving the received images to a different file.  Additional description of its usage is provided below.

- [generate_cartesian_shepp_logan_dataset.py](generate_cartesian_shepp_logan_dataset.py): Creates an MRD raw data file of a Shepp-Logan phantom with Cartesian sampling.  Borrowed from the [ismrmrd-python-tools](https://github.com/ismrmrd/ismrmrd-python-tools) repository.

- [dicom2mrd.py](dicom2mrd.py): This program converts a folder of DICOM images to an MRD image .h5 file, allowing DICOM images to be use as input for MRD streaming data.

- [mrd2dicom.py](mrd2dicom.py): This program converts an MRD image .h5 file to a folder of DICOM images.

- [mrd2gif.py](mrd2gif.py): This program converts an MRD image .h5 file into an animated GIF for quick previews.

There are several example "modules" that can be selected by specifying their name via the config (`-c`) argument:
- [invertcontrast.py](invertcontrast.py): This module accepts both incoming raw data as well as image data.  The image contrast is inverted and images are sent back to the client.

- [simplefft.py](simplefft.py): This module contains code for performing a rudimentary image reconstruction from raw data, consisting of a Fourier transform, sum-of-squares coil combination, signal intensity normalization, and removal of phase oversampling.

- [analyzeflow.py](analyzeflow.py): This module accepts velocity phase contrast image data and performs basic masking.

- [report.py](report.py): This module provides an example of generating report from a dictionary of parameters (keys) and their corresponding values.  An image with a text table is returned to the client and values are stored in the MetaAttributes to allow for batch scripted parsing.

### 2.1 <a name='Jsonconfig'></a>Additional (JSON) config
It is often useful for a client to provide additional configuration parameters during runtime to a module without changing code in the module itself.  For example this could be used to tune filter parameters, toggle additional outputs, or control optional processing steps.  The [client.py](client.py) is configured to look for a config file in the current folder, matching the name of the module and ending in `.json`, e.g. `invertcontrast.json` if the config module is named `invertcontrast`.  The example [invertcontrast.json](invertcontrast.json) can be modified with the `options` parameter set to `roi` to add an example ROI, `colormap` to add a color lookup table, and `rgb` to return an RGB image.

##  3. <a name='Savingincomingdata'></a>Saving incoming data
It may be desirable for the MRD server to save a copy of incoming data from the client.  For example, if the client is an MRI scanner, then the saved data can be used for offline simulations at a later time.  This may be particularly useful when the MRI scanner client is sending image data, as images are not stored in a scanner's raw data file and would otherwise require offline simulation of the MRI scanner reconstruction as well.

The feature may be turned on by starting the server with the ``-s`` option (disabled by default).  Data files are named by the current date/time and stored in ``/tmp/share/saved_data``.  The saved data folder can be changed using the ``-S`` option.  For example, to turn on saving of incoming data in the ``/tmp`` folder, start the server with:
```
python main.py -s -S /tmp
```

The ``-s`` flag is enabled in the startup script [start-fire-python-server-with-data-storage.sh](start-fire-python-server-with-data-storage.sh).

Alternatively, this feature can be enabled on a per-session basis when the client calls the server with the config ``savedataonly``.  In this mode, incoming data (raw or image) is saved, but no processing is done and no images are sent back to the client.

The resulting saved data files are in MRD .h5 format and can be used as input for ``client.py`` as detailed above.