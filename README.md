# Running Device Management Client example on the PSoC® 64 Secure Boot Wi-Fi BT Pioneer Kit (CYTFM_064B0S2_4343W)

![cypress-pioneer-kit](./img/cypress-pioneer-kit.png "Cypress Pioneer Kit")

This document guides you through all of the steps required to run Device Management Client example on the CYTFM_064B0S2_4343W target.

- [Prerequisites](#prerequisites).
- [Importing the example in Mbed Studio](#importing-the-example-in-mbed-studio).
- [Installing Cypress tools](#installing-cypress-tools).
- [Provisioning the device with initial credentials](#provisioning-the-device-with-initial-credentials)
- [Generating and provisioning Device Management credentials](#generating-and-provisioning-device-management-credentials).
- [Building and running the example](#building-and-running-the-example).
- [Updating firmware](#updating-firmware).

## Prerequisites

- [Arm Mbed Studio](https://os.mbed.com/studio/) for building and flashing the application.
- Install the `libusb` dependency for pyOCD based on the [Cypress documentation](https://www.cypress.com/file/502721/download#page=19&zoom=100,96,382).

    **Note:** Due to a known issue, Cypress recommends using [`libusb` version 1.0.21](https://github.com/libusb/libusb/releases/tag/v1.0.21) on Windows instead of the most recent version.
- OpenSSL toolkit (only if you do not have your own root CA private key and certificate and need to generate them yourself).
     - For Windows, use the [SLP distribution](https://slproweb.com/products/Win32OpenSSL.html) and set the `PATH` environment variable to point to the installation.


## Importing the example in Mbed Studio

[Import](https://os.mbed.com/docs/mbed-studio/current/create-import/index.html#importing-a-program-from-a-url) the code repository to Mbed Studio:

![mbed-studio-import](./img/mbed-studio-import.png "Cypress Pioneer Kit")

## Installing Cypress tools

1. From the **Mbed Studio** menu bar, select **Terminal** > **New Terminal** to open a terminal.

**Important Note** Due to a known issue with the Mbed Studio terminal feature, if you close and re-open Mbed Studio, you need to close the terminal tab and re-open it. Otherwise the terminal's path environment variable may be corrupted. 

1. Install CySecureTools:

    ```
    pip install cysecuretools
    ```

## Provisioning the device with initial credentials

You need to carry out this step only once on each board to be able to re-provision later with your root CA and device certificate.

1. In the Mbed Studio terminal, set up your project workspace for CySecureTools and create keys for provisioning:

    ```
    cd ./mbed-os-example-pelion-armdevsummit/mbed-os/targets/TARGET_Cypress/TARGET_PSOC6/TARGET_CYTFM_064B0S2_4343W
    ```

    ```
    cysecuretools -t cy8ckit-064b0s2-4343w init
    ```

    ```
    cysecuretools -t cy8ckit-064b0s2-4343w -p policy/policy_multi_CM0_CM4_tfm.json create-keys
    ```

    You will be prompted to overwrite existing files. Type `y` to continue.

1. Unplug your device from the power supply.
1. Remove the jumper shunt from J26.
1. Plug-in power.
1. Press the Mode button until the LED is always on to put the device in DAPLink mode.
1. To provision the board with basic configuration, run:

    ```
    cysecuretools -t cy8ckit-064b0s2-4343w -p policy/policy_multi_CM0_CM4_tfm.json provision-device
    ```

    ![provisioning](./img/provisioning.gif "Provisioning")

1. Unplug your device from the power supply.
1. Put back the jumper shunt in J26.
1. Plug-in power.

    **Note:** You don't need the keys and other files that are created in this flow in the future. At this point, you can delete these files.

For more information about the initial provisioning process, please see the ["Provision the Device" section of the CY8CKIT-064B0S2-4343W PSoC 64 Secure Boot Wi-Fi BT Pioneer Kit Guide](https://www.cypress.com/file/502721/download#page=30&zoom=100,96,382).

## Generating and provisioning Device Management credentials

1. Navigate to the root of the project and enter the `TARGET_CYTFM_064B0S2_4343W` directory:

    ```
    cd mbed-os-example-pelion-armdevsummit/TARGET_CYTFM_064B0S2_4343W
    ```

1. Create a `certificates` directory:

    ```
    mkdir certificates
    ```

1. Name your own root CA private key and certificate `rootCA.key` and `rootCA.pem` respectively, and place them in the `certificates/` directory.

    Alternatively, if you don't have a root CA, you can generate a root CA private key and certificate using the [OpenSSL toolkit](https://www.openssl.org/):

    ```
    openssl ecparam -out certificates/rootCA.key -name prime256v1 -genkey
    ```

    Create a new configuration file called `root.cnf` and place it in the `certificates/` directory. The file is passed as input to the `openssl` tool in the next step, to create the final certificate.
    The contents of the file should be the following:

    ```
    [ req ]
    distinguished_name=dn
    prompt = no
    [ ext ]
    basicConstraints = CA:TRUE
    keyUsage = digitalSignature, keyCertSign, cRLSign
    [ dn ]
    CN = ROOT_CA
    ```

    Finally, create the certificate:
    ```
    openssl req -key certificates/rootCA.key -new -x509 -days 7300 -sha256 -out certificates/rootCA.pem -config certificates/root.cnf -extensions ext
    ```

1. Upload the root CA certificate generated from the previous step to the portal. Refer to the [Pelion Documentation](https://www.pelion.com/docs/device-management/latest/provisioning-process/managing-ca-certificates.html#uploading-a-ca-certificate-or-certificate-chain) for instructions on how to accomplish this.

    **Important:** When you upload your root CA certificate to Device Management Portal, you must select **Enrollment - I received this certificate from the device manufacturer or a supplier** from the **How will devices use this certificate?** dropdown.

    ![certificate-upload](./img/certificate.gif "Certificate Upload")

1. Set up your project workspace for CySecureTools and create keys based on the [`cytfm_pelion_policy.json`](https://github.com/cvasilak/mbed-os-example-pelion-armdevsummit/blob/armdevsummit/TARGET_CYTFM_064B0S2_4343W/policy/cytfm_pelion_policy.json) policy:

    ```
    cysecuretools -t cy8ckit-064b0s2-4343w init
    cysecuretools -t cy8ckit-064b0s2-4343w -p policy/cytfm_pelion_policy.json create-keys
    ```

    > **Note:** You use these keys to sign future application images and the device uses the keys to verify the application images. Therefore, if you lose the keys, you need to re-provision the board with new keys.

1. Provision the device with your root CA, app keys and device certificate:
    ```
    python ../mbed-os/targets/TARGET_Cypress/TARGET_PSOC6/TARGET_CYTFM_064B0S2_4343W/reprov_helper.py -d cy8ckit-064b0s2-4343w -p policy/cytfm_pelion_policy.json -existing-keys --serial <device's unique serial number> -y
    ```
    Give your device any serial number that you like.

    ![provision-step2](./img/provisioning-step2.gif "Provision")

## Building and running the example

1. Navigate back to the root of the project:

    ```
    cd ..
    ```

1. In Windows only, rename `.mbedignore-for-win` to `.mbedignore`:

    ```
    rename .mbedignore-for-win .mbedignore
    ```

    > Due to [mbed-os issue 7129](https://github.com/ARMmbed/mbed-os/issues/7129), the include path might exceed the maximum Windows command line string length. Using `.mbedignore` decreases the length of the include path but makes these features unavailable:
    >* Certificate enrollment.
    >* Device Sentry.
    >* Secure device access.
    >* Factory flow using FCU.

1. In Windows, Mac and other case-insensitive file systems, apply a patch that resolves an issue with conflicting file names by running:

    ```
    cd mbed-cloud-client
    git am ..\mcc-fix-conflict-fname.patch
    cd ..
    ```

1. Add your WiFi access point information:

    1. Open the file `mbed_app.json`.

    1. Edit the lines to update the default WiFi SSID & password:

    ```
    "nsapi.default-wifi-ssid"                   : "\"SSID\"",
    "nsapi.default-wifi-password"               : "\"PASSWORD\"",
    ```

1. To enable firmware update:

    1. Install manifest-tool v2.0 or higher:

        ```
        pip install --upgrade manifest-tool
        ```
        The Cypress update flow requires the newest version of the manifest-tool.

    1. Initialize the environment:

        ```
        manifest-dev-tool init --force -a [access key from Device Management Portal]
        ```

        ![manifest-dev-tool-init](./img/manifest-dev-tool-init.gif "Manifest")

        For information about access keys and how you can create one, please see our [documentation](https://www.pelion.com/docs/device-management/latest/user-account/application-access-keys.html).

        ![access-keys](./img/access-key.gif "Access Keys")

1. Configure the target:

    Mbed OS supports two target configurations for this board. The default is `CY8CKIT_064B0S2_4343W`. For this tutorial, we will use `CYTFM_064B0S2_4343W` which enables support for Trusted Firmware-M (TF-M) secure services.

    1. In Mbed Studio, click the down arrow in the **Target** box.

    1. Click the chip icon to open the **Manage Custom Targets** menu.

    1. From the **USB device** dropdown, select the connected board.

    1. Enter `CYTFM_064B0S2_4343W` for in the **Target name** and **Build target** fields.

    1. Click **Save All**.

    ![select target](./img/target-selection.gif "Select Target")

1. Click the hammer icon (<img src="img/hammer.png" height="30">) to build the application:

    ![compile](./img/compile.gif "Compile")

    Upon success, the output looks like this:

    ```
    020-09-16 15:41:57,783 : C : INFO  : Image for slot BOOT signed successfully! (BUILD/CYTFM_064B0S2_4343W/ARMC6\tfm_s.hex)
    2020-09-16 15:41:58,168 : C : INFO  : Image for slot UPGRADE signed successfully! (BUILD/CYTFM_064B0S2_4343W/ARMC6\tfm_s_upgrade.hex)
    2020-09-16 15:42:00,372 : C : INFO  : Image for slot BOOT signed successfully! (BUILD/CYTFM_064B0S2_4343W/ARMC6\mbed-os-example-pelion-armdevsummit.hex)
    2020-09-16 15:42:01,813 : C : INFO  : Image for slot UPGRADE signed successfully! (BUILD/CYTFM_064B0S2_4343W/ARMC6\mbed-os-example-pelion-armdevsummit_upgrade.hex)
    | Module                                        |      .text |    .data |       .bss |
    |-----------------------------------------------|------------|----------|------------|
    | TARGET_CYTFM_064B0S2_4343W\cy_factory_flow.o  |   2113(+0) |    0(+0) |   2000(+0) |
    | [lib]\c_w.l                                   |  13006(+0) |   16(+0) |    392(+0) |
    | [lib]\fz_wm.l                                 |   1888(+0) |    0(+0) |      0(+0) |
    | [lib]\libcpp_w.l                              |      5(+0) |    0(+0) |      0(+0) |
    | [lib]\libcppabi_w.l                           |     44(+0) |    0(+0) |      0(+0) |
    | [lib]\m_wm.l                                  |    754(+0) |    0(+0) |      0(+0) |
    | anon$$obj.o                                   |     48(+0) |    0(+0) |   5120(+0) |
    | main.o                                        |   3059(+0) |    0(+0) |    369(+0) |
    | mbed-cloud-client\device-sentry-client        |    166(+0) |    0(+0) |      0(+0) |
    | mbed-cloud-client\factory-configurator-client |  11190(+0) |    2(+0) |     83(+0) |
    | mbed-cloud-client\mbed-client                 |  67037(+0) |   22(+0) |     12(+0) |
    | mbed-cloud-client\mbed-client-pal             |  14254(+0) |    1(+0) |     79(+0) |
    | mbed-cloud-client\source                      |  10166(+0) |    5(+0) |     24(+0) |
    | mbed-cloud-client\update-client-hub           |  28753(+0) |  146(+0) |   5178(+0) |
    | mbed-os\connectivity                          | 165477(+0) |  200(+0) | 105681(+0) |
    | mbed-os\drivers                               |   6614(+0) |    0(+0) |     84(+0) |
    | mbed-os\events                                |   2138(+0) |    0(+0) |   4672(+0) |
    | mbed-os\features                              |   6470(+0) |    1(+0) |    228(+0) |
    | mbed-os\hal                                   |   3580(+0) |    8(+0) |    248(+0) |
    | mbed-os\platform                              |  10393(+0) |   80(+0) |   1420(+0) |
    | mbed-os\rtos                                  |  16028(+0) |  168(+0) |   7897(+0) |
    | mbed-os\storage                               |   5873(+0) |    0(+0) |    548(+0) |
    | mbed-os\targets                               | 508073(+0) |  533(+0) |   3817(+0) |
    | update_default_resources.o                    |    432(+0) |    0(+0) |      0(+0) |
    | Subtotals                                     | 877561(+0) | 1182(+0) | 137852(+0) |
    Total Static RAM memory (data + bss): 139034(+0) bytes
    Total Flash memory (text + data): 878743(+0) bytes
    Update Image: BUILD/CYTFM_064B0S2_4343W/ARMC6\mbed-os-example-pelion-armdevsummit_update.bin
    Image: BUILD/CYTFM_064B0S2_4343W/ARMC6\mbed-os-example-pelion-armdevsummit.hex
    ```

    > **Notes:**
    * Depending on your system, the building can take two minutes (best case) to 20 minutes (worst case).
    * Ignore this error:
        ```
        Configuration error: Bootloader not supported on this target. RAM start not found in targets.json.
        ```

1. Click the run icon (<img src="img/run.png" height="25">) to flash and run the application:

    Alternatively, you can flash the application directly:
    1. Drag and drop the hex output file (`BUILD/CYTFM_064B0S2_4343W/ARMC6/mbed-os-example-pelion-armdevsummit.hex`) to the mounted drive for the board.
    1. Reset the board.

1. Open the serial monitor (click **View** > **Serial Monitor** in Mbed Studio) and choose baud rate `115200`.

1. Enroll the device to your Pelion account:

    The first time the application successfully boots up and connects to the network, it prints its enrollment ID on the serial monitor.

    ![enrollment](./img/enrollment.png "Enrollment ID")

    1. Copy the enrollment ID from the terminal output.
    1. Upload the enrollment ID to Device Management:
        1. In Device Management Portal, click **Device directory** > **Enrolling devices**.
        1. Click the **+ Enroll devices** button.
        1. Enter the enrollment ID in the **Device enrollment key** field.
        1. Click **Add single device**.
    1. Go back to the terminal, press **C** to continue, and check that the device connects to Device Management.

    ![enrollment](./img/enroll.gif "Enrollment ID")

> **Note:** For development purposes, you can reset the Device Management credentials by running `pyocd erase -s 0x101C0000-0x101C9000`.

## Updating firmware

The `CYTFM_064B0S2_4343W` target board has two cores - CM0 for the T-FM firmware, CM4 for the example application.

We currently support updating the example application in the CM4 core.

**To update the example application:**

1. Update the firmware version in the `cytfm_pelion_policy.json` file:

    1. Go to `"id": 16` in the file.

       This section of the file holds the CM4 core configuration.

    1. Update `"version": "<new firmware version>",`.

        Where `<new firmware version>` is a string in MSB.LSB (Most Significant Byte/Least Significant Byte) format.

1. Click the hammer icon (<img src="img/hammer.png" height="30">) to build the upgraded signed image:

    This creates a `./BUILD/CYTFM_064B0S2_4343W/ARMC6/mbed-os-example-pelion-armdevsummit_upgrade.hex` file.

    The manifest tool does not currently support hex files; therefore, you must convert the image to bin format.

1. Convert the upgrade image from hex to bin format:

    ```
    python inthex2bin.py BUILD/CYTFM_064B0S2_4343W/ARMC6/mbed-os-example-pelion-armdevsummit_upgrade.hex
    ```
    This creates the `./BUILD/CYTFM_064B0S2_4343W/ARMC6/mbed-os-example-pelion-armdevsummit_upgrade.bin` file.

1. Perform the update:

    ```
    manifest-dev-tool update-v1 --payload-path BUILD/CYTFM_064B0S2_4343W/ARMC6/mbed-os-example-pelion-armdevsummit_upgrade.bin --fw-version <new firmware version> --device-id <device ID> --start-campaign --wait-for-completion --no-cleanup --timeout 3600
    ```

    Where:

    - `<new firmware version>` is a 64-bit unsigned integer, where 32 MSBs represent the major version and 32 LSBs represent the minor version. For example, version 1.0 is represented as `4294967296` and version 1.1 as `4294967297`. You can run the following to calculate the value, replacing `1.0` with your desired version number:
    
    ```
    python -c "a, b = '1.0'.split('.'); print(int(a)<<32 + int(b))"
    ```
    - `<device ID>` is the ID of the device to be updated.

    ![starting-update-campaign](./img/starting_update_campaign.gif "Manifest")

    When executing this command, the manifest tool:

    * Uploads the update image to the Device Management update service.
    * Creates and signs a manifest file with the digest of the update image and its Device Management URL.
    * Uploads that manifest to the Device Management update service.
    * Creates an update campaign using the manifest and a default device filter matching the device ID.
    * Starts the update campaign.
    * Waits for the update campaign to finish.

    You can validate that the device is using the upgraded firmware in the serial monitor printout:

    ```
    Current FW image version: <new firmware version>
    ```

    ![update program](./img/update.gif "Update Program")


    You can check the status of the update campaign by selecting it in the portal and opening its details pane. It should be marked as successful:

    ![view-update-campaign-details](./img/update-campaigns.png "View Update Campaign Details")


## Troubleshooting Tips

- If the Python commands fail to run
    - Ensure that you are calling commands from within a **Mbed Studio** terminal.
    - If you close Mbed Studio, close and re-open the terminal within Mbed Studio. This resets it to a known good state.
    - To check that the terminal environment is set correctly, type `which python` (Mac or Linux) or `where python` on windows. The path to the correct version of Python should be within _Mbed Studio_ files.  

- If the provisioning tools fail to connect 
    - Ensure that the USB cable is plugged into the KitProg3 USB (left side of board)
    - Check that the KitProg3 status LED is blinking (for DAPLink mode)
    - If you are on Windows, ensure that the correct version of _libusb_ is installed
    - See [Secure Boot SDK User Guide](https://www.cypress.com/file/480976/download) for more tips on correct installation

- If the provisioning process fails 
    - Ensure that jumper J26 is off for the first-time provisioning, then back on for provisioning thereafter. 
    - If the provisioning process fails before completion, you may encounter problems when trying it again. If this occurs, try adding the **--erase-boot** option to the _cysecurtools_ command. 

- If the application fails to build in Mbed Studio
    - Double check that the build target CYTFM_064B0S2_4343W is set correctly. 
    - Make sure that you **do not** update the libaries.
    - Ensure there are no formatting errors in mbed_app.json
    
- If the device fails to connect to the Pelion service
    - Double check that you have added your WiFi credentials correctly
    - Make sure that you have uploaded the correct root CA certificate (.pem file) to the Pelion console. If not correct, the device can appear to have a network error.
    - Problems can occur if you run `manifest-dev-tool init` within a project multiple times, which causes the update certificate to change. If this occurs, follow these steps. 
    1. Flash the device with the new image with _new_ update certificate
    1. Then run pyocd command to erase the internal storage, `pyocd erase -s 0x101C0000-0x101C9000`.  
    1. Reset the board. THe new update certificate will be saved.

- You missed getting the enrollemnt ID from the device
    - If you accidentally press `c` in the serial monitor before copying the enrollment ID, run the command `pyocd erase -s 0x101C0000-0x101C9000` to start over.
    - If you have trouble copying the enrollment ID from the serial monitor, please note that there is a known issue in Mbed Studio in which highlighted text is not visibly noticable. However, if you select the text and copy it, you should be able to paste it.

- If the firmware update campaign fails to start
    - If a campaign is launched when there is an existing campaign ongoing that targets the same device(s), then the new campaign can get stuck in "daft" mode. 
    - To stop an ongoing campaign, go to the Pelion device management portal, select Firmware update > Update campaigns, click the active campaign, then click **Stop**.

- If the device does not start downloading the update
    - Ensure that the firmware version is set correctly in the correct section of **cytfm_pelion_policy.json**.
    - Ensure that the firmware version is newer than the version that is currently running on the board. 
    - Make sure to convert the update hex file to binary and point the manifest-dev-tool to the correct file. 
    - Ensure the Device ID and Access Key are correct and coorespond to the device that is enrolled to the same account that your access key comes from.
