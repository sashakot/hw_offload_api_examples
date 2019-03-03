# Understanding Erasure Coding Offload

This document describes the Reed-Solomon Erasure Coding hardware offload feature supported on Mellanox BlueField™ and ConnectX®-5 adapter family ( ConnectX®-4 provides only limited functionality)

Audience is advanced users and developers.

## Reference

[Reed Solomon Tutorial: Reed Solomon Encoding Example Case](https://www.youtube.com/watch?v=jgO09opx56o)
[Swift Object Storage: Adding Erasure Codes](http://www.snia.org/sites/default/files/Luse_Kevin_SNIATutorialSwift_Object_Storage2014_final.pdf)
[RDMA Aware Networks Programming User Manual](http://www.mellanox.com/related-docs/prod_software/RDMA_Aware_Programming_user_manual.pdf)
[OFVWG Erasure Coding RDMA Offload](https://downloads.openfabrics.org/ofv/erasure-coding.pdf)

## What is Reed-Solomon Erasure Coding?

Erasure coding is a mathematical method to *encode data* in a way that it can be recovered in case of disk failures.

For those without any background on storage recovery, it is advised to watch this video and this presentation that explains at high level the algorithm and supplies examples and illustrations. 


## Erasure Coding Offload Programming Models

There are different programming models that an application can choose to implement RAID and Erasure Coding (EC) offload. We will show an example of 5-2 coding (5 data blocks and 2 calculated redundancy blocks).

Erasure Coding and Decoding hardware offload is supported by Mellanox ConnectX-5 adapters.(ConnectX-4 provides only limited functionality)

### Software Calculations (No offload)

Most applications today perform Erasure Coding calculations in software then send the data and redundancy blocks to the relevant nodes/disks (OSDs).

In these cases the calculations are performed by the CPU and, therefore, there is no hardware offload. CPU utilization in this case will be high as well as IO operations.

### Hardware Offload

Using Mellanox ConnectX-4 adapters, Erasure Coding calculations can be offloaded to the adapter's ASIC.

There are several models of hardware offload as described in below.

#### Synchronous Encode Calculations

Synchronous EC calculation can be used by existing applications simply by replacing a JErasure-like API to use the hardware driven EC calculation.

##### Programming Steps

1. Call encode_sync(data, code, block_size) API – blocking.
2. Send the data buffers to the corresponding nodes (no ordering dependency on step 1).
3. Once encode_sync() returns, send the code buffers to the corresponding nodes.

The advantages here are:

- Easy code conversion to work with a hardware driven EC calculation
- CPU utilization savings as the HCA offloads the EC calculation

Note: Latency and/or message-rate and/or bandwidth are less likely to improve directly.

#### Asynchronous Encode Calculations

Asynchronous EC calculation is also possible where the application can post an EC calculation operation and get an event notification when the calculation is done. This model allows the application to become more efficient as at the time it is waiting for an EC calculation, it is free to compute or execute or service any other job.

When the adapter completes the calculation, it notifies the application so it can continue to send the data and coding blocks to the relevant peers (OSDs).


##### Programming Steps

1. Post encode_async(data, code, block_size, …).
2. Send the data buffers to the corresponding nodes (no ordering dependency on step 1).
3. Wait for encode_async to complete (asynchronously), free to continue with tasks execution meanwhile.
4. Send the code buffers to the corresponding nodes.

Unlike the synchronous model, the call to async encoding is non-blocking and is a fast operation. The calling thread provides a done() function pointer for post-calc execution and continues with other compute operations. It may, for instance, post the sending of the data blocks at this stage. Once the calculation is completed, the done() function passed by the calling thread is executed. The done() function is expected to trigger the sending of the code blocks.

Asynchronous Encode and Send Calculations

Generally, EC is used to spread an object store across multiple nodes (OSDs) in a cluster -- this operation is called striping an object. Applications using this model will be able to post a compound job that includes the entire striping operation to the adapter and continue to execute the next task. The adapter will offload both the EC calculation and send the corresponding blocks to the corresponding nodes.

rograming Steps:

Post a compound operation encode_send(data, code, block_size, nodes, …).
Free to continue with tasks execution.
The completion of the EC calculation is implicit. The entire transaction is considered as completed after all the individual data transfer operations are completed. The user might want to signal the corresponding SEND operations or use any other way that guarantees that the data-transfer is completed successfully and stored in the storage media.

The advantage of this method is reduced latency and message rates (IO operations) as the application saves a SW interrupt and cache-line bounces (due to completion context execution) which exists in the former two models. However, the conversion of existing applications to work in a fully offloaded striping operation is less trivial.






Requirments:
gf-complete
Jerasure

Build:
./build-ec-encode-example.sh

Examples:
ibv_ec_encoder -i mlx5_0 -k 2 -m 1 -w 4 -D /tmp/EcEncodeDecode_2_1 -s 1024 -v
ibv_ec_decoder -i mlx5_0 -k 2 -m 1 -w 4 -D /tmp/EcEncodeDecode_2_1 -C /tmp/EcEncodeDecode_2_1.code.offload -s 1024 -E 1,0 -v
ibv_ec_encoder_async -i mlx5_0 -k 2 -m 1 -w 4 -D /tmp/EcEncodeDecode_2_1 -s 1024 -l 16 -v
ibv_ec_updater -i mlx5_0 -k 8 -m 2 -w 4 -D /tmp/EcUpdate_8_2 -C /tmp/EcUpdate_8_2.code.offload -s 1024 -u 0,0,0,1,0,0,0,1 -c 1,0 -v
./run_ec_perf_encode.sh -d mlx5_0 -i ib0 -k 20 -m 9 -w 8 -r 60 -c 24 -b 1024 -q 1 -l 64 -a
