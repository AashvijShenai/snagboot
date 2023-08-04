# A Developer's Guide to Snagboot

## Introduction
Snagboot is comprised of Snagrecover which is the recovery tool and Snagflash which is the flashing tool

## Prerequisties
https://github.com/AashvijShenai/snagboot/blob/main/docs/board_setup.md

## Directory Tree

│   .gitignore
│   CONTRIBUTING.md
│   fw.yaml
│   install.sh
│   LICENSE
│   MANIFEST.in
│   pyproject.toml
│   README.md
│
├───.github
│   └───workflows
│           main.yml
│
├───docs
│       board_setup.md
│       developers_guide.md
│       fw_binaries.md
│       snagflash.md
│       snagrecover.md
│       troubleshooting.md
│       tutorial_snagrecover.cast
│       tutorial_snagrecover.gif
│
└───src
    ├───snagflash
    │   │   cli.py
    │   │   dfu.py
    │   │   fastboot.py
    │   │   ums.py
    │   │   utils.py
    │   │   __init__.py
    │   │
    │   └───bmaptools
    │           BmapCopy.py
    │           BmapCreate.py
    │           BmapHelpers.py
    │           Filemap.py
    │           __init__.py
    │
    └───snagrecover
        │   50-snagboot.rules
        │   am335x_usb_setup.sh
        │   cli.py
        │   config.py
        │   supported_socs.yaml
        │   utils.py
        │   __init__.py
        │
        ├───firmware
        │   │   am335x_fw.py
        │   │   firmware.py
        │   │   imx_fw.py
        │   │   ivt.py
        │   │   rom_container.py
        │   │   sama5_fw.py
        │   │   samba_applet.py
        │   │   __init__.py
        │   │
        │   └───sunxi_fw
        │           fel-to-spl-thunk.bin
        │           fel-to-spl-thunk.S
        │           mmu.py
        │           soc_info.yaml
        │           sunxi_fw.py
        │           __init__.py
        │
        ├───protocols
        │       bootp.py
        │       dfu.py
        │       fastboot.py
        │       fel.py
        │       hab_constants.py
        │       imx_sdp.py
        │       memory_ops.py
        │       sambamon.py
        │       __init__.py
        │
        ├───recoveries
        │       am335x.py
        │       am62x.py
        │       imx.py
        │       sama5.py
        │       stm32mp1.py
        │       stm32_flashlayout.py
        │       sunxi.py
        │       __init__.py
        │
        └───templates
                am335x-beaglebone-black.yaml
                am62x-beagle-play.yaml
                imx28-evk.yaml
                imx6-colibri-imx6ull.yaml
                imx6-var-som-mx6.yaml
                imx7-colibri-imx7d.yaml
                imx8-dart-mx8m-mini.yaml
                imx8qm-mek.yaml
                imx93-evk.yaml
                sama5-sama5d2xplained.yaml
                sama5-sama5d3xplained.yaml
                sama5-sama5d4xplained.yaml
                stm32mp1-stm32mp135f-dk.yaml
                stm32mp1-stm32mp157f-dk2.yaml
                sunxi-orangepi-pc.yaml

## Snagrecover
https://github.com/AashvijShenai/snagboot/blob/main/docs/snagrecover.md

Command: `snagrecover -s {soc_models} -f {templates.yaml}`

snagrecover calls cli()

cli()
    - parse arguments

    - intialize global config (init_config) into dictionary recovery_config
        - check_soc_model -> look up supported_soc.yaml
        - get_family
        - get usb vid:pid from default_usb_ids using soc_family OR parse it from argument
        - store firmware in recovery_config["firmware"] fw_configs{}
    
    - call recovery am62x_recovery() (snagrecover/recovers/am62x.py/main)
        - get_usb(vid:pid) ->utils.py
            - call usb.core.find (USB core function)
            - get_active_configuration (USB core function)

        - run_firmware:tiboot3 -> snagrecover/firmware/firmware.py : am62x_run
            - find altsetting(partition name) (tiboot3:bootloader, tispl:tispl.bin, u-boot:u-boot.img) [which shows up when `dfu-util -l` is run]
            - dfu.search_partid(dev, partname) ->snagrecover/protocols/dfu.py
                - get_active_configuration (USB core function)
                - intfs = cfg.interfaces() : iterate over altsettings of the same interface to match partname

            - dfu.DFU(dev, stm32=False):class that implements dfu
                - get_active_configuration (USB core function)
                - set transfer size (default: 1024) by checking for extra_descriptors

            - download_and_run(fw_blob, partid, offset=0, size=len(fw_blob))
                - DFU stuff

            - if fw==tispl, detach              FIX: change detach to uboot

        - should re-enumerate usb device
        - dispose_resources
        - get_usb(vid:pid)
        - run_firmware:u-boot           FIX order
        - run_firmware:tispl
