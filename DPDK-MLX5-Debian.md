# Configure DPDK on Mellanox ConnectX-5 Ex

### Important Parameters
- Server series - Dell PowerEdge R6515; Linux 5.10
- NIC - Mellanox ConnectX-5 EX 2_Ports 40/100GbE QSFP28-Adapter, PCIe Low Profile
- DPDK 20.08 [Official Guide](https://doc.dpdk.org/guides-20.08/nics/mlx5.html)
    - Require Firmware version: **16.21.1000** and above.
    - Mellanox OFED **4.5** and above / Mellanox EN version: **4.5** and above

### Installation Record

#### 1. Prerequisites of installing DPDK on MLX5

`sudo apt-get update -y`

1. libibverbs-dev `sudo apt-get install libibverbs-dev`
2. libmlx5-dev `sudo apt-get install libmlx5-1`
3. kernel modules: `mlx5_core`

#### 2. MLNX_OFED V5.1 （[official webpage](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)）

```cpp
$ hostnamectl
Operating System: Debian GNU/Linux 11 (bullseye)                                                        
Kernel: Linux 5.10.103.1.amd64-smp                                                      
Architecture: x86-64
```

So chose [MLNX_OFED_LINUX-5.6-2.0.9.0-debian11.2-x86_64.tgz](https://content.mellanox.com/ofed/MLNX_OFED-5.6-2.0.9.0/MLNX_OFED_LINUX-5.6-2.0.9.0-debian11.2-x86_64.tgz)

- `tar -xf`
- cd &&  `./mlnxofedinstall --upstream-libs --dpdk`
- `sudo modprobe -a ib_uverbs mlx5_core mlx5_ib`

#### 3. Firmware Update 

Since the firmware in our hw is given by Dell (not NVIDIA), fw cannot be updated automatically. Check Dell support.
- Useful links: DELL [firmware](https://www.dell.com/support/home/en-ie/drivers/driversdetails?driverid=ymp89&lwp=rt), [R6515](https://www.dell.com/support/home/en-is/drivers/driversdetails?driverid=9pnf6&oscode=us008&productcode=poweredge-r6515), [User manual](https://dl.dell.com/manuals/all-products/esuprt_data_center_infra_int/esuprt_data_center_infra_network_adapters/mellanox-adapters_user's-guide_en-us.pdf) of mlx5 for Dell EMC Power Edge
- Procedures: Download `.BIN` and `chmod +x` and `./.....BIN` to execute

#### 4. Make DPDK 20.08
 - Modify dpdk/config/common_base
   ```shell
   CONFIG_RTE_LIBRTE_MLX5_PMD=y 
   CONFIG_RTE_KNI_KMOD=y
   ``` 
 - Set env var `export RTE_SDK=$(pwd)`, `export RTE_TARGET=x86_64-native-linux-gcc`  
 - Run `dpdk/usertools/dpdk-setup.sh` choose `[41] x86_64-native-linux-gcc`
 - Using **rte_kni.ko** driver. Choose `[47] insert kni module` or manually `sudo insmod <target>/build/kmod/rte_kni.ko`.  
 - Add hugepages reservation, check NUMA (`[49] Setup hugepage mappings for NUMA systems`), 1024

#### 5. Test
A loopback cable is needed for connecting two ports. 
 - `ibv_devinfo` => `state PORT_ACTIVE(4)`
 - Run `[55] testpmd`. You should see stats of received/sent pkts after `start` and `stop`.
 
**Note: Mellanox NICs don't require binding drivers to ports explicitly using `dpdk-devbind.py`**. Even though it is neccessary to bind (UIO/igb_uio) for Intel NIC. 

#### Misc
- `kni_dev.h` bugs happened to LINUX >= 5.9.0. [Solution](https://inbox.dpdk.org/dev/1599652785-42385-1-git-send-email-humin29@huawei.com/t/)
  ```cpp
  #if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 9, 0)
    ret = get_user_pages_remote(tsk->mm, iova, 1,
				    FOLL_TOUCH, &page, NULL, NULL);
  #else
    ret = get_user_pages_remote(tsk, tsk->mm, iova, 1,
 				    FOLL_TOUCH, &page, NULL, NULL);
  #endif 
  ```
