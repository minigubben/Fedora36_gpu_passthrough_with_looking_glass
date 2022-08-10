# prerequisites

Virtualization tools
```bash
sudo dnf install @virtualization
```

Looking glass build dependencies Fedora 35+. See https://looking-glass.io/wiki/Installation_on_other_distributions
```bash
sudo dnf install cmake gcc gcc-c++ libglvnd-devel fontconfig-devel spice-protocol make nettle-devel \
            pkgconf-pkg-config binutils-devel libXi-devel libXinerama-devel libXcursor-devel \
            libXpresent-devel libxkbcommon-x11-devel wayland-devel wayland-protocols-devel \
            libXScrnSaver-devel libXrandr-devel dejavu-sans-mono-fonts
```

# Setting up PCI passthrough

Make sure the GPU and it's audio device is in it's own iommu group.
Grab the device ID and audio device ID for the GPU you want to pass through from the GPU checker. ex. `10de:1b81` and `10de:10f0`

Add the drivers to your initramfs by creating the following file /etc/dracut.conf.d/vfio.conf
```
add_drivers+=" vfio vfio_iommu_type1 vfio_pci vfio_virqfd "
```

Rebuild the initramfs

```bash
sudo dracut -f
```

Add the following to the line GRUB_CMDLINE_LINUX in /etc/default/grub
```
amd_iommu=on amd_iommu=pt rd.driver.pre=vfio-pci vfio-pci.ids=<GPU:ID>,<GPUAUDIO:ID>
```

You can also passthrough a whole usb host device if you wish by adding
```
pci-stub.ids=<USB_ID>
```

Regenerate grub config by running 
```bash
sudo grub2-mkconfig -o /etc/grub2-efi.cfg
```
And reboot.



Make sure the GPU and audio interface is using vfio-pci by running 
```bash
lspci -v
```


# Configuring VM
Create the VM with Virtual Machine Manager as usual but on the last step select "Cystomize configuration before install"
In the configuration make sure you select chipset "Q35" and firmware "OVMF_CODE.fd"
Remove all displays and graphics devices and then click add and select the graphic card and it's audio interface for PCI passthrough.
You should pass through a keyboard and mouse for inital setup.

Now you can start the machine.

# Building looking glass client
Download the latest source from https://looking-glass.io/downloads Don't clone from git as you wont get all dependencies.

To build and install the client run the following commands.
```bash
mkdir client/build
cd client/build
cmake ../
make
make install
```

# Configuring VM for looking glass

Add the following to the device section in the VM config. XML edit can be enabled in settings.
```
<shmem name="looking-glass">
  <model type="ivshmem-plain"/>
  <size unit="M">64</size>
  <address type="pci" domain="0x0000" bus="0x10" slot="0x01" function="0x0"/>
</shmem>
```
Now you must install the IVSHMEM drivers in windows. you can get them from https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/ and it needs to be version 0.1.161 or above.
Open device manager and expand "System Devices" install new drivers for "PCI standard RAM Controller". You cant just select the driver folder directly but must instead browse and select the ivshmem.inf manually.

Looking glass needs permissions to the shared memory file, you are supposed to be able to give this automatically acording to https://looking-glass.io/docs/B5.0.1/install/#client-install but i never got it working and just change permissions manually every time i reboot.
To do this run 
```bash
sudo chown 107:1000 /dev/shm/looking-glass
sudo chmod 664 /dev/shm/looking-glass
```

To add mouse and keyboard support click add hardware inside your vm settings and select graphics->spice server. To prevent the spice server from breaking your passthrough you need to select the created video device and modify the XML to the following
```
<model type='none'/>
```
Now remove the tablet and add new mouse mouse and keyboard devices. 


To get clipboard synchronization to work you should add the following config to you VMs device configuration.
```
<channel type="spicevmc">
  <target type="virtio" name="com.redhat.spice.0"/>
  <address type="virtio-serial" controller="0" bus="0" port="1"/>
</channel>

```

```
<channel type="spicevmc">
  <target type="virtio" name="com.redhat.spice.0"/>
  <address type="virtio-serial" controller="0" bus="0" port="1"/>
</channel>
```

In windows you now need to install the looking glass host from https://looking-glass.io/downloads
Once this is done everything should work. Run looking-glass-client to start the client.

# Good to know
If you dont get a title bar you can move the window by pressing the super key and dragging it with the mouse with the mouse.
Hold scroll lock to see shortcuts. Scroll lock + F to toggle fullscreen.
