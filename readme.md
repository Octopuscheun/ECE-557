# Project: SHA3——AWS——FPGA
**Name:** Trevor(Yuzhe) Zhang 
**NetID:** yz972
**course:** ECE 557
**date:** 3/13/2025

## Important Information:
For this lab, the testbench has been modified to accommodate the results from the C implementation. Before verifying functionality, the hash array in the testbench should be replaced accordingly to ensure correct comparison with the expected output.

The top-level file for this implementation is sha3_beethoven.scala. The entire design is structured into three phases,
Read data, SHA3, And Output data. more detailed design information are as follows.

## Project introduction:
This lab focuses on implementing an accelerator for the SHA-3 algorithm on an AWS F2 FPGA instance. The core idea is to design a finite state machine (FSM) to manage the processes of reading input data, performing SHA-3 computation, and outputting the results. The system is structured into five stages:

# s_init – Initialization Stage

The system waits for io.req.fire(valid && ready), signaling the start of an operation. Once the request is acknowledged, it transitions to the s_read_data stage.

# s_read_data – Data Reading Stage

In this stage, 17 data elements are read into the message buffer, which is directly connected to the SHA-3 Core Input Vector. When the message buffer is fully populated (message_index == 17.U), the system transitions to the s_sha3 stage.

# s_sha3 – SHA-3 Computation Stage

The SHA-3 computation is performed in this stage, leveraging the implementation from the previous lab. Once the hash computation completes (hash.valid && hash.ready), the output data is stored in the hash buffer, and the system moves to the s_write_data stage.

# s_write_data – Data Writing Stage

The computed hash is output through vec_out_data.data.bits. A total of 4 data elements are written, and upon completion (hash_index == 4.U), the system transitions to the s_done stage.

# s_done – Completion Stage

This stage marks the completion of the SHA-3 acceleration process. When io.resp.fire(ready && valid), the system resets and returns to the s_init stage, ready for the next operation.

The following table provides detailed information on the design’s ports and their functionalities.

**Inputs and Outputs declaration**
**Port Name**|**Input/output**|**Description**
| ---------- | -------------- | --------------- | 
|vec_addr|Input|Provide the read & write address|
|io.req.ready|Output|Start signal for the system|
|io.req.valid|Input|Start signal for the system|
|io.resp.ready|Input|End signal for the system|
|io.resp.valid|Output|End signal for the system|

**Val Name**|**Description**
| ---------- | --------------- | 
|ReaderModuleChannel|    |
|vec_in_request.bits.addr|Receive the read address from vec_addr|
|vec_in_request.bits.len|Specify the read data length|
|vec_in_request.ready(output)|Ready signal for request data|
|vec_in_request.valid(input)|Valid signal for request data|
|vec_in_data.data.bits|Data channel|
|vec_in_data.data.ready(Output)|Ready signal for Receiving data|
|vec_in_data.data.valid(Input)|valid signal for Receiving data|

**Val Name**|**Description**
| ---------- | --------------- | 
|WriteModuleChannel|    |
|vec_out_request.bits.addr|Receive the write address from vec_addr|
|vec_out_request.bits.len|Specify the write data length|
|vec_out_request.ready(Output)|Ready signal for requesting sending data|
|vec_out_request.valid(Input)|Valid signal for requesting sending data|
|vec_out_data.data.bits|Data channel|
|vec_out_data.data.ready(Input)|Ready signal for sending data|
|vec_out_data.data.valid(Output)|Valid signal for sending data|

### Design Validation:

A testbench is implemented to verify the correctness of the module. In the testbench design, each data element is assigned as a UInt64_t type. Therefore, the data copy_to and copy_back processes must be adapted to accommodate this specific data type. As a result, modifications are required in the memcmd process to ensure compatibility with the design. The basic implementation is shown below:

```
void dma_workaround_copy_to_fpga(remote_ptr &q) {
    uint64_t sz = q.getLen() / 8;
    auto * intar = (uint64_t *)q.getHostAddr();
    for (uint64_t i = 0; i < sz; ++i) {
        auto ptr = q + i * 8;
        uint32_t lower = intar[i];
        uint32_t upper = intar[i] >> 32;
        DMAHelper::memcmd(0, q + i * 8, lower, 1).get();
        DMAHelper::memcmd(0, q + i * 8 +4, upper,1).get();
    }
}
void dma_workaround_copy_from_fpga(remote_ptr &q) {
    auto * intar = (uint64_t*)q.getHostAddr();
    for (uint64_t i = 0; i < 4; ++i) {
        auto ptr = q + i * 8;
        auto resp_low = DMAHelper::memcmd(0, q + i * 8, 0, 0).get();
        auto resp_upp = DMAHelper::memcmd(0, q + i * 8  + 4, 0, 0).get();
        intar[i] = ((uint64_t)resp_upp.payload << 32) | resp_low.payload;
    }
}

```
To verify the functionality of the system, we run simulations on both the AWS F2 FPGA instance and a local machine. Each data element is initialized to 0, and after computation, the program successfully outputs the expected results. This confirms that the entire design functions correctly.
