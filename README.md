# GPU Passthrough Vega iGPU on a Ryzen 3 4350 with VEGA 6 Renoir Architecture or Any other Vega iGPU 5700G, 5600G

Finally Figured Out how to make the iGPU run as a passthrough GPU!

here some one has describe what he had done to make it work:

https://forum.proxmox.com/threads/amd-ryzen-5600g-igpu-code-43-error.138665/

kind of followed these steps as well!

1. vendor-reset
  1. clone this repo: https://github.com/gnif/vendor-reset
  2. find out the vendor id: lspci -nkk
  3. add your one (renoir is 1636, cezanne is 1638) to the src/device-db.h file in the vendor-reset folder under AMD_NAVI10:
      85 #define _AMD_NAVI10(op) \
      86     {PCI_VENDOR_ID_ATI, 0x1636, op, DEVICE_INFO(AMD_NAVI10)}, \
      87     {PCI_VENDOR_ID_ATI, 0x7310, op, DEVICE_INFO(AMD_NAVI10)}, \
      88     {PCI_VENDOR_ID_ATI, 0x7312, op, DEVICE_INFO(AMD_NAVI10)}, \
  4. install it with dkms install .
  5.  I had to do this to make the install run under DEbian 12 with kernel 6.12
    	•	Edit `src/amd/amdgpu/atom.c`
	    •	Change line 32: From:
       #include <asm/unaligned.h>
        to
       #include <linux/unaligned.h>
  6. then add the module (its actually called vendor_reset not with -!) to the /etc/module
  7. Restart
  8. Luckily my mainboard has my gpu already on a separate iommu group, otherwise you would have to separate your iommu group with sth. like asm google it and you will find what I mean
  9. Find out how to extract your vbios rom and the hdmi audio rom from either the GPU or an UEFI update File.
     If you have the same cpu/gpu as me - 4350g or a 5700g - you can use the dat files from here.
     1636 is the vega 6 of the 4350G and 1638 is the vega 8 of the 5700G.
  	 both use the same audio device so the ATIAudioDevice_AA01.rom file for the HDMI audio device can be used for both!
     IMPORTANT: You must include the audio device otherwise the passthrough will end as the famous error 43.

	 To extract my files from the bios update file: 
     I did use this one here: https://winraid.level1techs.com/t/tool-guide-news-uefi-bios-updater-ubu/3035 called UBU-1.80 and a UEFI File
      Actually just extracted it and then used UBU.cmd which asked for my UEFI Update file.
	then you need to convert it as well:
	
 	https://github.com/isc30/ryzen-gpu-passthrough-proxmox?tab=readme-ov-file#configuring-the-gpu-in-the-windows-vm
	https://github.com/isc30/ryzen-gpu-passthrough-proxmox/discussions/18#discussioncomment-8627679

  10. Add the vbios to the folder /usr/share/vgabios nothing else works for kvm under Debian!
  11. Add the PCIe Devices to your domain.xml and define the rom file:
     ```xml
     <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
      </source>
      <rom file="/usr/share/vgabios/vbios_1636.dat"/>
      <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <driver name="vfio"/>
      <source>
        <address domain="0x0000" bus="0x06" slot="0x00" function="0x1"/>
      </source>
      <rom file="/usr/share/vgabios/ATIAudioDevice_AA01.rom"/>
      <address type="pci" domain="0x0000" bus="0x09" slot="0x00" function="0x0"/>
    </hostdev>
    ```
  11. UEFI Necessary! so use sth. like this: 
  12. no grub options necessary! iommu is used per default!
  13. install the amd drivers. actually for me the official ones from amd work!
  14. Well yeah if you follow this path you will have a working igpu passthrough
  15. 

  another important step which was necessary for the 5700G:

  adding a reset_gpu.bat script as logon
  and a disable_gpu.bat script on shutdown/restart
