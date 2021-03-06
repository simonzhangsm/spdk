# SPDK vhost Test Plan

## Current Tests

### Integrity tests

#### vhost self test
- compiles SPDK and Qemu
- launches SPDK Vhost
- starts VM with 1 NVMe device attached to it
- issues controller "reset" command using sg3_utils on guest system
- performs data integrity check using dd to write and read data from the device
- runs on 3 host systems (Ubuntu 16.04, Centos 7.3 and Fedora 25)
  and 1 guest system (Ubuntu 16.04)
- runs against vhost scsi and vhost blk

#### FIO Integrity tests
- NVMe device is split into 4 LUNs, each is attached to separate vhost controller
- FIO uses job configuration with randwrite mode to verify if random pattern was
  written to and read from correctly on each LUN
- runs on Fedora 25 and Ubuntu 16.04 guest systems
- runs against vhost scsi and vhost blk

#### Lvol tests
- starts vhost with at least 1 NVMe device
- starts 1 VM or multiple VMs
- lvol store is constructed on each NVMe device
- on each lvol store 1 lvol bdev will be constructed for each running VM
- Logical volume block device is used as backend instead of using
  NVMe device backend directly
- after set up, data integrity check will be performed by FIO randwrite
  operation with verify flag enabled
- optionally nested lvols can be tested with use of appropriate flag;
  On each base lvol store additional lvol bdev will be created which will
  serve as a base for nested lvol stores.
  On each of the nested lvol stores there will be 1 lvol bdev created for each
  VM running. Nested lvol bdevs will be used along with base lvol bdevs for
  data integrity check.
- runs against vhost scsi and vhost blk

#### Filesystem integrity
- runs SPDK with 1 VM with 1 NVMe device attached.
- creates a partition table and filesystem on passed device, and mounts it
- runs Linux kernel source compilation
- Tested file systems: ext2, ext3, ext4, brtfs, ntfs, fat
- runs against vhost scsi and vhost blk

#### Windows HCK SCSI Compliance Test 2.0.
- Runs SPDK with 1 VM with Windows Server 2012 R2 operating system
- 4 devices are passed into the VM: NVMe, Split NVMe, Malloc and Split Malloc
- On each device Windows HCK SCSI Compliance Test 2.0 is run

#### MultiOS test
- start 3 VMs with guest systems: Ubuntu 16.04, Fedora 25 and Windows Server 2012 R2
- 3 physical NVMe devices are split into 9 LUNs
- each guest uses 3 LUNs from 3 different physical NVMe devices
- Linux guests run FIO integrity jobs to verify read/write operations,
    while Windows HCK SCSI Compliance Test 2.0 is running on Windows guest

#### vhost hot-remove tests
- removing NVMe device (unbind from driver) which is already claimed
    by controller in vhost
- hotremove tests performed with and without I/O traffic to device
- I/O traffic, if present in test, has verification enabled
- checks that vhost and/or VMs do not crash
- checks that other devices are unaffected by hot-remove of a NVMe device
- performed against vhost blk and vhost scsi

#### vhost scsi hot-attach and hot-detach tests
- adding and removing devices via RPC to a controller which is already in use by a VM
- I/O traffic generated with FIO read/write operations, verification enabled
- checks that vhost and/or VMs do not crash
- checks that other devices in the same controller are unaffected by hot-attach
    and hot-detach operations

### Vhost initiator test
Testing vhost initiator with fio write, randwrite, rw and randrw with verificiation enabled.
Tests include vhost-user (vhost initiator connecting to the socket on the same machine)
and virtio-pci (virtio initiator in a VM connecting to the virtual PCI SCSI device created by the hypervisor)
All tests are run in virtio-user mode. Tests 2-3, 5-9 are additionally run in virtio-pci mode.

#### Test Case 1 - vhost initiator test with malloc
1. Run vhost with one scsi controller and with one malloc bdev with 512 block size.
2. Prepare config for bdevio with virtio section.
3. Run bdevio test application with config.
4. Generate the fio config file given the list of all bdevs.
5. Run fio tests: iodepth=128, block_size=4k, rw, randwrite, write, randrw with verification.
6. Check if fio tests are successful.

#### Test Case 2 - vhost initiator test with nvme
1. Run vhost with one scsi controller and with one nvme bdev with 512 block size.
2. Repeat steps 2-6 from test case 1.

#### Test Case 3 - vhost initiator test with lvol
1. Run vhost with one scsi controller and with one lvol bdev with 512 block size.
2. Repeat steps 2-6 from test case 1

#### Test Case 4 - vhost initiator test with malloc
1. Run vhost with one scsi controller and with one malloc bdev with 4096 block size.
2. Repeat steps 2-6 from test case 1.

#### Test Case 5 - vhost initiator test with lvol
1. Run vhost with one scsi controller and with one lvol bdev with 4096 block size.
2. Repeat steps 2-6 from test case 1

#### Test Case 6 - vhost initiator test with nvme disk (size larger than 4G)
1. Run vhost with one scsi controller and with one nvme bdev with 512 block size and disk size larger than 4G
   to test if we can read, write to device with fio offset set to 4G.
2. Repeat steps 4-6 from test case 1.

#### Test Case 7 - vhost initiator test with multiqueue
1. Run vhost with one scsi controller (one malloc bdev and one nvme bdev).
2. Generate the fio config file given the list of all bdevs.
3. Run fio tests: iodepth=128, block_size=4k, rw, randread, randwrite, read, write, randrw with verify
4. Check if fio tests are successful.

#### Test Case 8 - vhost initator test with multiple socket
1. Run vhost with two scsi controllers, one with nvme bdev and one with malloc bdev.
2. Generate the fio config file given the list of all bdevs.
3. Run fio tests: iodepth=128, block_size=4k, write with verification.
4. Check if fio tests are successful.

#### Test Case 9 - vhost initiator test with unmap
1. Run vhost with one controller and one nvme bdev with 512 block size.
2. Run fio test with sequential jobs: trim, randtrim, write.
   All this jobs run with verification enabled.
   Use trim_verify_zero fio option to check if blocks are returned as zeroes.
   Write with verify after trim to check if we still can write and read from device.
3. Check if fio test ends with success.
4. Repeat steps 1-3 on host for malloc with 4096 block size and 512 block size.

### Live migration
Live migration feature allows to move running virtual machines between SPDK vhost
instances.
Following tests include scenarios with SPDK vhost instances running on both the same
physical server and between remote servers.
Additional configuration of utilities like SSHFS share, NIC IP address adjustment,
etc., might be necessary.

#### Test case 1 - single vhost migration
- Start SPDK Vhost application.
    - Construct a single Malloc bdev.
    - Construct two SCSI controllers and add previously created Malloc bdev to it.
- Start first VM (VM_1) and connect to Vhost_1 controller.
  Verify if attached disk is visible in the system.
- Start second VM (VM_2) but with "-incoming" option enabled, connect to.
  Connect to Vhost_2 controller. Use the same VM image as VM_1.
- On VM_1 start FIO write job with verification enabled to connected Malloc bdev.
- Start VM migration from VM_1 to VM_2 while FIO is still running on VM_1.
- Once migration is complete check the result using Qemu monitor. Migration info
  on VM_1 should return "Migration status: completed".
- VM_2 should be up and running after migration. Via SSH log in and check FIO
  job result - exit code should be 0 and there should be no data verification errors.
- Cleanup:
    - Shutdown both VMs.
    - Gracefully shutdown Vhost instance.

#### Test case 2 - single server migration
- Detect RDMA NICs; At least 1 RDMA NIC is needed to run the test.
  If there is no physical NIC available then emulated Soft Roce NIC will
  be used instead.
- Create /tmp/share directory and put a test VM image in there.
- Start SPDK NVMeOF Target application.
    - Construct a single NVMe bdev from available bound NVMe drives.
    - Create NVMeoF subsystem with NVMe bdev as single namespace.
- Start first SDPK Vhost application instance (later referred to as "Vhost_1").
    - Use different shared memory ID and CPU mask than NVMeOF Target.
    - Construct a NVMe bdev by connecting to NVMeOF Target
        (using trtype: rdma).
    - Construct a single SCSI controller and add NVMe bdev to it.
- Start first VM (VM_1) and connect to Vhost_1 controller. Verify if attached disk
  is visible in the system.
- Start second SDPK Vhost application instance (later referred to as "Vhost_2").
    - Use different shared memory ID and CPU mask than previous SPDK instances.
    - Construct a NVMe bdev by connecting to NVMeOF Target. Connect to the same
      subsystem as Vhost_1, multiconnection is allowed.
    - Construct a single SCSI controller and add NVMe bdev to it.
- Start second VM (VM_2) but with "-incoming" option enabled.
- Check states of both VMs using Qemu monitor utility.
  VM_1 should be in running state.
  VM_2 should be in paused (inmigrate) state.
- Run FIO I/O traffic with verification enabled on to attached NVME on VM_1.
- While FIO is running issue a command for VM_1 to migrate.
- When the migrate call returns check the states of VMs again.
  VM_1 should be in paused (postmigrate) state. "info migrate" should report
  "Migration status: completed".
  VM_2 should be in running state.
- Verify that FIO task completed successfully on VM_2 after migrating.
  There should be no I/O failures, no verification failures, etc.
- Cleanup:
    - Shutdown both VMs.
    - Gracefully shutdown Vhost instances and NVMEoF Target instance.
    - Remove /tmp/share directory and it's contents.
    - Clean RDMA NIC / Soft RoCE configuration.

#### Test case 3 - remote server migration
- Detect RDMA NICs on physical hosts. At least 1 RDMA NIC per host is needed
  to run the test.
- On Host 1 create /tmp/share directory and put a test VM image in there.
- On Host 2 create /tmp/share directory. Using SSHFS mount /tmp/share from Host 1
  so that the same VM image can be used on both hosts.
- Start SPDK NVMeOF Target application on Host 1.
    - Construct a single NVMe bdev from available bound NVMe drives.
    - Create NVMeoF subsystem with NVMe bdev as single namespace.
- Start first SDPK Vhost application instance on Host 1(later referred to as "Vhost_1").
    - Use different shared memory ID and CPU mask than NVMeOF Target.
    - Construct a NVMe bdev by connecting to NVMeOF Target
        (using trtype: rdma).
    - Construct a single SCSI controller and add NVMe bdev to it.
- Start first VM (VM_1) and connect to Vhost_1 controller. Verify if attached disk
  is visible in the system.
- Start second SDPK Vhost application instance on Host 2(later referred to as "Vhost_2").
    - Construct a NVMe bdev by connecting to NVMeOF Target. Connect to the same
      subsystem as Vhost_1, multiconnection is allowed.
    - Construct a single SCSI controller and add NVMe bdev to it.
- Start second VM (VM_2) but with "-incoming" option enabled.
- Check states of both VMs using Qemu monitor utility.
  VM_1 should be in running state.
  VM_2 should be in paused (inmigrate) state.
- Run FIO I/O traffic with verification enabled on to attached NVME on VM_1.
- While FIO is running issue a command for VM_1 to migrate.
- When the migrate call returns check the states of VMs again.
  VM_1 should be in paused (postmigrate) state. "info migrate" should report
  "Migration status: completed".
  VM_2 should be in running state.
- Verify that FIO task completed successfully on VM_2 after migrating.
  There should be no I/O failures, no verification failures, etc.
- Cleanup:
    - Shutdown both VMs.
    - Gracefully shutdown Vhost instances and NVMEoF Target instance.
    - Remove /tmp/share directory and it's contents.
    - Clean RDMA NIC configuration.

### Performance tests
Tests verifying the performance and efficiency of the module.

#### FIO Performance 6 NVMes
- SPDK and created controllers run on 2 CPU cores.
- Each NVMe drive is split into 2 Split NVMe bdevs, which gives a total of 12
  in test setup.
- 12 vhost controllers are created, one for each Split NVMe bdev. All controllers
  use the same CPU mask as used for running Vhost instance.
- 12 virtual machines are run as guest systems (with Ubuntu 16.04.2); Each VM
  connects to a single corresponding vhost controller.
  Per VM configuration is: 2 pass-through host CPU's, 1 GB RAM, 2 IO controller queues.
- NVMe drives are pre-conditioned before the test starts. Pre-conditioning is done by
  writing over whole disk sequentially at least 2 times.
- FIO configurations used for tests:
    - IO depths: 1, 8, 128
    - Blocksize: 4k
    - RW modes: read, randread, write, randwrite, rw, randrw
    - Write modes are additionally run with 15 minute ramp-up time to allow better
    measurements. Randwrite mode uses longer ramp-up preconditioning of 90 minutes per run.
- Each FIO job result is compared with baseline results to allow detecting performance drops.

## Future tests and improvements

### Stress tests
- Add stability and stress tests (long duration tests, long looped start/stop tests, etc.)
to test pool
