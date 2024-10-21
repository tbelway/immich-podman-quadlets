---
sidebar_position: 100
---

# HardWare Acceleration setup for immich with podman

If you are using NVIDIA on a redhat OS such as fedora or almalinux, reboots sometimes cause the nvidia-uvm and nvidia-uvm-tools to just... not load. Use the following script and systemd file to correct this behaviour:

You can ultimately place this wherever you want as long as it's referenced appropriately in the systemd file:
nv_init.sh
```bash
#!/bin/bash

echo "modprobe nvidia"
/sbin/modprobe nvidia

echo "modprobe nvidia-uvm"
/sbin/modprobe nvidia-uvm

if [ "$?" -eq 0 ]; then
  # Count the number of NVIDIA controllers found.
  NVDEVS=`lspci | grep -i NVIDIA`
  N3D=`echo "$NVDEVS" | grep "3D controller" | wc -l`
  NVGA=`echo "$NVDEVS" | grep "VGA compatible controller" | wc -l`

  N=`expr $N3D + $NVGA - 1`
  echo "mknode nvidia"
  for i in `seq 0 $N`; do
    if [ ! -e /dev/nvidia$i ]; then
      mknod -m 666 /dev/nvidia$i c 195 $i
    fi
  done

  echo "mknode nvidiactl"
  if [ ! -e /dev/nvidiactl ]; then
    mknod -m 666 /dev/nvidiactl c 195 255
  fi
else
  echo "exit NVIDIA controller"
  exit 1
fi

if [ "$?" -eq 0 ]; then
  # Find out the major device number used by the nvidia-uvm driver
  D=`grep nvidia-uvm /proc/devices | awk '{print $1}'`

  echo "mknode nvidia-uvm"
  if [ ! -e /dev/nvidia-uvm ]; then
    mknod -m 666 /dev/nvidia-uvm c $D 0
  fi
  echo "mknode nvidia-uvm-tools"
  if [ ! -e /dev/nvidia-uvm-tools ]; then
    mknod -m 666 /dev/nvidia-uvm-tools c $D 0
  fi
else
  echo "exit nvidia-uvm"
  exit 1
fi

echo "generate yaml in case of driver upgrade"
nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

/etc/systemd/system/init_nvidia.service
```bash
[Unit]
Description=NVIDIA Init for CUDA
After=network.target
ConditionPathExists=|!/dev/nvidia
ConditionPathExists=|!/dev/nvidiactl
ConditionPathExists=|!/dev/nvidia-uvm
ConditionPathExists=|!/dev/nvidia-uvm-tools

[Service]
Type=oneshot
ExecStart=/bin/bash /opt/git/containers/rootless/containers/immich/nv_init.sh
RemainAfterExit=yes

[Install]
WantedBy=default.target
```
