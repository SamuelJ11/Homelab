# Homelab

## 1. OVERVIEW

This project uses a Makefile which can selectively build targets.

The default target for 'make' is the executable for task 1 (PA2_Task1), so simply running 'make' will build this target (you could also explicitly specify the target by running 'make 1' to build PA2_Task1).

To build the executable for task 2 (PA2_Task2) run 'make 2'.

While the Makefile handles all dependencies, it is helpful to know that Task 1 depends on the following headers and sources:

    • Task1.c           (main source)
    • TimeInterval.c    (auxiliary function for calculating timeout values)
    • TimeInterval.h    (declaration of TimeInterval.c)

And Task 2 depends on the following headers and sources:

    • Task2.c           (main source)
    • TimeInterval.c    (auxiliary function for calculating timeout values)
    • TimeInterval.h    (header file for TimeInterval.c) 
    • Packet.c          (implementation for the packet "class" that handles serializing and deserializing structs)
    • Packet.h          (declaration of Packet.c)  

Once both executables have been built, feel free to run 'make clean' to remove unnecessary object files.

## 2. EXECUTION

### Task 1

Task 1 has several optional parameters specified with default values to minimize the number of arguments the user must supply to the program.

Run the following command on the server:

    • ./PA2_Task1 server  

Then in a separate or split terminal, run the following command on the client:

    • ./PA2_Task1 client

The following are the default parameters that were implicitly passed:

    • server_ip = 127.0.0.1     argv[2]
    • server_port = 12345       argv[3]
    • num_threads = 4           argv[4]
    • num_requests = 1000000    argv[5]
    • pipeline_size = 4         argv[6]

These values can be changed by running the program with the additional command-line arguments specified in that exact order.

The output of running the code using the default parameters gives this output on my Codespace:

    ==============================================================
    Results For Thread 1: 

    Total packets sent: 1000000 
    Total packets received: 999743 
    Number of packets lost: 257
    ==============================================================
    Results For Thread 2: 

    Total packets sent: 1000000 
    Total packets received: 999701 
    Number of packets lost: 299
    ==============================================================
    Results For Thread 3: 

    Total packets sent: 1000000 
    Total packets received: 999508 
    Number of packets lost: 492
    ==============================================================
    Results For Thread 4: 

    Total packets sent: 1000000 
    Total packets received: 999585 
    Number of packets lost: 415
    ==============================================================
    ==============================================================
    Summary Statistics: Finished Processing 4000000 Total Requests 

    Average Packet Loss Rate Across All Threads: 0.0366 %
    Average Timeout Interval Across all Threads: 573 µs
    ==============================================================

### Task 2

Task 2 can be run exactly like Task 1, and has the optional parameters specified with the following default values:

    • server_ip = 127.0.0.1     argv[2]
    • server_port = 12345       argv[3]
    • num_threads = 4           argv[4]
    • num_requests = 40000      argv[5]
    • pipeline_size = 4         argv[6]

The output of running the code using the default parameters gives this output on my Codespace:

    ==============================================================
    Results For Thread 1: 

    Total packets sent: 40044 
    Total packets received: 40044 
    Number of packets lost: 0
    Number of retransmissions: 44
    ==============================================================
    Results For Thread 2: 

    Total packets sent: 40020 
    Total packets received: 40020 
    Number of packets lost: 0
    Number of retransmissions: 20
    ==============================================================
    Results For Thread 3: 

    Total packets sent: 40036 
    Total packets received: 40036 
    Number of packets lost: 0
    Number of retransmissions: 36
    ==============================================================
    Results For Thread 4: 

    Total packets sent: 40028 
    Total packets received: 40028 
    Number of packets lost: 0
    Number of retransmissions: 28
    ==============================================================
    ==============================================================
    Summary Statistics: Finished Processing 160128 Total Requests 

    Average Packet Loss Rate Across All Threads: 0.0000 %
    Average Timeout Interval Across all Threads: 295 µs
    Total Number of Retransmissions Across All Threads: 128
    ==============================================================

## 3. CISCO MODELLING LABS (CML)

The code for checking whether a packet's given timeout value is within an acceptable range is given below (lines 143 - 154):

    • if (timeout_µs < (10 * num_threads * pipeline_size))
        {
            /* Restore timeout and update the timespec struct that epoll_pwait2() actually uses */
            timeout_µs = (10 * num_threads * pipeline_size);
            packettiming.TimeoutInterval.tv_nsec = timeout_µs * 1000; /* timespec struct uses nanoseconds */
        }
        /* If timeout exceeds 1000x minimum threshold, something has gone seriously wrong */
        if (timeout_µs > (10000 * num_threads * pipeline_size)) 
        {
            puts("server is overloaded or not responding; maximum timeout exceeded");
            exit(1);
        }

This code checks whether the time it took for the client to receive a response from the server is within a certain range.

Here the minimum timeout value is in microseconds (µs), and if the program is run with the default parameters (4 threads, pipeline size of 4), then the minimum timeout is calculated as:

    • 4 x 4 x 10 = 160 µs

Similarly the maximum timeout value is simply 1000x this minimum value:

    • 160 x 1000 = 160000 µs = 0.16 s

Every time a timeout occurs, the timeout value will double (temporarily) to avoid a premature timeout for the next packet (lines 176 - 184):

    •  else if (nfds == 0) /* timeout (packet loss) */
        {
            /* Double the TimeoutInterval (temporarily) to avoid a premature timeout for the next packet */
            timeout_µs *= 2;

            /* Update the timespec struct that epoll_pwait2() actually uses */
            packettiming.TimeoutInterval.tv_sec = timeout_µs / 1000000;
            packettiming.TimeoutInterval.tv_nsec = (timeout_µs % 1000000) * 1000;
        }

If too many consecutive timeouts occur, the timeout value will keep doubling until eventually the maximum timeout value is reached.

This will lead the client to assume something has gone wrong with the connection, as denoted by the following error message:

    • server is overloaded or not responding; maximum timeout exceeded

If encountering this error message, sometimes re-running the program once or twice will resolve the issue.

If this doesn't work, re-run the program and try halving one of the last three parameters supplied to the program:

For example, you could try:

    • ./PA2_Task1 client 127.0.0.1 12345 4 1000000 2    (halved pipeline size)
    • ./PA2_Task1 client 127.0.0.1 12345 2 1000000 4    (halved thread count)
    • ./PA2_Task1 client 127.0.0.1 12345 4 500000 4    (halved number of requests)

Additionally, this approach can be adopted for Task 2 if necessary.

The hardware specifications of your device or whether or not the program is being run in a native Linux environment will greatly impact the likelihood of this error occurring.

## 4. REMARKS

Task 2, while functional, is not completely robust and, dare I say, correct.

In the run_server() function, there is a line of code that is as follows:
    
    • server_packet.expected_seqnum = client_packet.next_seqnum + 1; (line 414) 

Then later it just echoes this information back to the client:

    • SerializeServer(&server_packet, echo_buf);            (line 420)                 
    • sentMsgSize = sendto(UDPSock, echo_buf,  . . . );     (line 421) 

The glaring issue here is that the server isn't actually "expecting" anything specific. 

If Thread 1 sends packets 0, 1, 2, 3 in a burst, then:

    Server receives 0, updates expected_seqnum to 1

    Server receives 1, updates expected_seqnum to 2
                .
                .
                .

If the ACKs are delayed and the client times out, the client resets its loop sliding window to its base:

    • i = client_packet.base;   (in client_thread_func() line 217)

Then the send loop for the client repeats and the following code is executed:

    • client_packet.next_seqnum = i;                                    (line 142)
    • sentMsgSize = sendto(packetdata->client_fd, send_buf, . . .);     (line 147)

Later after the client resends packet with sequence number 'i', the server receives 'i' again and does the following: 

    • server_packet.expected_seqnum = client_packet.next_seqnum + 1;    (line 414)

Essentially the server "acknowledges" any packet it sees, rather than waiting for the next expected one.

The correct behavior should be to simply discard packet with sequence number 'i' and wait for the next in-order packet.

I bring this to the reader's attention because if we recall the default parameter for num_requests for this task:

    • num_requests = 40000      argv[5]

This value had to be set relatively low, otherwise a "runaway" scenario was observed where the control condition of the client loop:

    • while (client_packet.base < num_requests && !stop)

Will always be true and the loop never terminates unless the user interrupts the client program (ctrl + c).

I'm not sure if the issue described above is causing this "runaway" behavior or not, and I was not able to debug it in time.

If the grader has any suggestions as to what could be happening, please send me an email at 

    sjab225@uky.edu

Thank you for reading, and I look forward to any suggestions for improvement!