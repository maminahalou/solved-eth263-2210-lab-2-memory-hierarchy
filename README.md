Download Link: https://assignmentchef.com/product/solved-eth263-2210-lab-2-memory-hierarchy
<br>
In this lab, you will extend Lab 1 to implement a memory hierarchy that includes an L2 cache and a DRAM-based main memory. Similar to Lab 1, we will fully specify the cycle-by-cycle behavior of the L2 cache and DRAM.

While the L1 caches are tightly integrated with the pipeline, the L2 cache is more decoupled from the pipeline. In this lab, we will maintain the semantics for the L1 caches that were described in Lab 1, but replace the constant L1 cache miss latency of 50 cycles with a variable latency, which involves at least the L2 cache and may involve DRAM. Whenever the processor would have begun a 50 cycle L1 cache miss access stall in Lab 1, it instead issues a request to the memory hierarchy in this lab. When the data is available from the L2 or main memory (as specified below), the memory hierarchy issues a <em>cache fill </em>notification back to the L1 caches. If the L1 cache is still stalling on the missing data (i.e., no cancellation occurred), the data is inserted into the cache and the stall ends in the <strong>next cycle</strong>. We now specify exactly how the memory hierarchy is structured and how requests are handled in more detail.

<h1>1.                 Your Task: Extending the Simulator with L2 Cache and DRAM</h1>

<h2>1.1.       Unified L2 Cache</h2>

The unified L2 cache is accessed whenever there is a miss in either of the two L1 caches (instruction or data).

<strong>Organization. </strong>It is a <strong>16-way </strong>set-associative cache that is <strong>256 KB </strong>in size with <strong>32 byte </strong>blocks (this implies that the cache has <strong>512 sets</strong>). When accessing the cache, the set index is equal to [13:5] of the address.

<strong>MSHRs. </strong>The L2 cache has <strong>16 </strong>MSHRs (<em>miss-status holding registers</em>), which is more than enough for our simple in-order pipeline. For every L2 cache miss, the L2 cache allocates an MSHR that keeps track of the miss. An MSHR consists of three fields: <em>(i) </em>valid bit, <em>(ii) </em>address of the cache block that triggered the miss, and <em>(iii) </em>done bit. You can assume that all bits are initialized to 0. When main memory serves an L2 cache miss, it sends a fill notification to the L2 cache that sets the done bit in the corresponding MSHR (i.e., if done=1, then MSHR entry becomes invalid and valid bit=0). The valid bit can be used to indicate that an MSHR entry contains a valid memory address (and, for example, not initialized address in the MSHR).

<strong>Accessing the L2 Cache. </strong>Assume that the L2 cache has a large number of ports and that accesses to it are never serialized. When a memory access misses in either of the L1 caches, the L2 cache is “probed”<a href="#_ftn1" name="_ftnref1"><sup>[1]</sup></a> immediately in the <em>same </em>cycle as the L1 cache miss. Depending on whether the access hits or misses in the L2 cache, it is handled differently.

<ul>

 <li><strong>L2 Cache Hit. </strong>After <strong>15 cycles</strong>, the L2 cache sends a fill notification to the L1 cache (instruction or data). For example, if an access misses in an L1 cache but hits in the L2 cache at the 0th cycle, then the L1 cache receives a fill notification at the 15th cycle.</li>

 <li><strong>L2 Cache Miss. </strong>Immediately, the L2 cache allocates an MSHR for the miss. After <strong>5 cycles</strong>, the L2 cache sends a memory request to main memory (this models the latency between the L2 cache and the memory controller). When the memory request is served by main memory, the L2 cache receives a fill notification. After <strong>5 cycles</strong>, the retrieved cache block is inserted into the L2 cache and the corresponding MSHR is freed (this models the latency between the memory controller and the L2 cache). In the <strong>same cycle</strong>, if the pipeline is stalled because of the cache block, then the L2 cache sends a fill notification to the L1 cache. Also in the <strong>same cycle</strong>, the cache block is inserted into the L1 cache. In the <strong>next cycle</strong>, the pipeline becomes unstalled.</li>

</ul>

<strong>Management Policies. </strong>A new cache block is inserted into the L2 cache at the MRU position. For an L2 cache hit, a cache block is promoted to the MRU position. In the same cycle, there can be as many as two L2 cache hits: one from the fetch stage and another from the memory stage. If they happen to hit in two different cache blocks that belong to the same set, then there is an ambiguity about which block is promoted to the MRU position. In this case, we assume that the memory stage accesses the L2 cache before the fetch stage: we promote the block that is requested by the fetch stage to the MRU position and the block that is requested by the memory stage to the MRU-1 position.

The L2 cache adopts true LRU replacement.

<strong>Other.</strong>

<ul>

 <li>Assume that the L1 and L2 caches are initially empty.</li>

 <li>Assume that the program that runs on the processor <strong>never </strong>modifies its own code (referred to as self-modifying code): a given cache block <strong>cannot </strong>reside in both the L1 caches.</li>

 <li>Unlike an L1 cache access, an L2 cache access <strong>cannot </strong>be canceled once it has been initiated.</li>

 <li>Similar to Lab 1, we do <strong>not </strong>model dirty evictions that can occur from either the L1 data cache (to the L2 cache) or the L2 cache (to main memory). When a newly inserted cache block evicts a dirty cache block, we assume that the dirty cache block is written into the immediately lower level of the memory hierarchy <em>instantaneously</em>. Furthermore, this happens <strong>without causing any side effects</strong>: the LRU ordering of the cache blocks in the corresponding cache set is not affected; in DRAM, buses/banks are not utilized and the row buffer status does not change. When the L2 cache does not have any free MSHRs remaining (which should not happen in our simple in-order pipeline), then an L1 cache miss <strong>cannot </strong>even probe the L2 cache. In other words, a free MSHR is a prerequisite for an access to the L2 cache, regardless of whether the access would actually hit or miss in the L2 cache. A freed MSHR can be re-allocated to another L2 cache miss in the <strong>same cycle</strong>.</li>

 <li>Once a cache miss (data or instruction) occurs in the MEM stage, you are free to stall the pipeline until the request is served from L2 cache (hit or miss) or memory. However, if you have both an instruction cache miss and a data cache miss from an older instruction that is already in the MEM stage, then you can serve these two instructions by allocating one entry of MSHR per each miss.</li>

</ul>

<h2>1.2.       Main Memory (DRAM)</h2>

<strong>Organization. </strong>The DRAM system has a <strong>single rank </strong>on a <strong>single channel</strong>, where the channel consists of the command/address/data buses. The rank has <strong>eight banks</strong>. Each bank consists of <strong>64K rows</strong>, where each row is <strong>8KB </strong>in size. When accessing main memory, the bank index is equal to [7:5] of the address and the row index is equal to [31:16] of the address. For example, the 32-byte cache block at address 0x00000000 is stored in the 0th row of the 0th bank. As another example, the 32-byte cache block at address 0x00000020 is stored in the 0th row of the 1st bank.

<strong>Memory Controller. </strong>The memory controller holds the memory requests received from the L2 cache in a request queue. The request queue can hold an infinite number of memory requests. In each cycle, the memory controller scans the request queue (including the memory requests that arrived during the current cycle) to find the requests that are “schedulable” (to be described later). If there is only one schedulable request, the memory controller initiates the DRAM access for that request. If there are multiple schedulable requests, the memory controller prioritizes the requests in the following order, which is similar to the FR-FCFS policy:

<ol>

 <li>requests that are row buffer hits are prioritized over others</li>

 <li>requests that arrived earlier are prioritized over others</li>

 <li>requests coming from the memory stage are prioritized over others</li>

</ol>

<strong>DRAM Commands &amp; Timing. </strong>A DRAM access consists of a sequence of commands issued to a DRAM bank. There are four DRAM commands: ACTIVATE (issued with the bank/row addresses), READ/WRITE (issued with the column address), and PRECHARGE (issued with the bank address). Each command utilizes both the command and address buses for <strong>4 cycles</strong>. Once a DRAM bank receives a command, it becomes “busy” for <strong>100 cycles </strong>and cannot accept any other commands. 100 cycles after receiving a READ or WRITE command, the bank is ready to send/receive a 32 byte chunk of data over the data bus – this transfer utilizes the data bus for <strong>50 cycles</strong>.

<strong>Row-Buffer Status. </strong>Each DRAM bank has a row buffer. Depending on the status of the bank’s row buffer, a different sequence of commands is required to serve a memory request to the bank. For example, let us assume that a memory request needs to access the 0th row in the 0th bank. At this point, there are three possible scenarios:

<ul>

 <li>Row buffer hit: when the 0th row is already in the row buffer</li>

 <li>Row buffer miss: when there is no row loaded in the corresponding bank’s row buffer</li>

 <li>Row buffer conflict: when a row different from the 0th row is loaded in the corresponding bank’s row buffer</li>

</ul>

The following table summarizes the sequence of commands that is required for each of the three scenarios.

<table width="352">

 <tbody>

  <tr>

   <td width="140">Scenario</td>

   <td width="212">Commands (Latency)</td>

  </tr>

  <tr>

   <td width="140">Row-Buffer Hit</td>

   <td width="212">READ/WRITE</td>

  </tr>

  <tr>

   <td width="140">Row-Buffer Miss</td>

   <td width="212">ACTIVATE, READ/WRITE</td>

  </tr>

  <tr>

   <td width="140">Row-Buffer Conflict</td>

   <td width="212">PRECHARGE, ACTIVATE, READ/WRITE</td>

  </tr>

 </tbody>

</table>

<strong>“Schedulable” Request. </strong>A memory request is defined to be schedulable when all of its commands can be issued without any conflict on the command/address/data buses, as well as the bank, at the cycles when commands are to be issued and corresponding data is to be expected. For example, at the 0th cycle, a request to a bank with a closed row buffer is schedulable if and only if <strong>all </strong>of the following conditions are met:

<ol>

 <li>The command and address buses are free at cycles <em>0, 1, 2, 3, 100, 101, 102, </em>and <em>103</em></li>

 <li>The data bus us free during cycles <em>200-249</em></li>

 <li>The bank is free during cycles <em>0-99 </em>and <em>100-199</em></li>

</ol>

First, the command/address buses are free at cycles 0, 1, 2, 3, 100, 101, 102, 103. Second, the data bus is free during cycles 200–249. Third, the bank is free during cycles 0–99 and 100–199.

<strong>Other Assumptions and Policies.</strong>

<ul>

 <li>Assume that the row buffers of all the banks are initially closed, i.e., there is no row in any row buffer.</li>

 <li>The memory controller follows the open row policy, i.e., once a row is opened it is <em>not </em>closed unless another scheduled request requires the closing of the row to access another row in the same bank.</li>

</ul>

<h1>2.      Extra Credit</h1>

We will offer up to 25% extra credit <em>for this lab </em>for implementing an even more realistic system considering writebacks from the cache and refreshes in the memory controller. A fully correct implementation of Lab 2 is a prerequisite for extra credit.

You can add support for writebacks and refreshes in your simulator. The simulator with support for writebacks should model the writeback of dirty blocks from the L1 cache to the L2 cache, and from the L2 cache to the main memory. You need to ensure these requests occupy resources and take time, similarly to other requests.

The simulator with support for refreshes should model periodic refresh of every single DRAM row. These refreshes should be issued by the memory controller. The refresh rate should be configurable, e.g., a refresh to a row is issued every N cycles. You need to accurately model the contention caused by refreshes to other requests and ensure refresh requests occupy request queue entries in the memory controller, and take time. For this extra-credit assignment, we refer you to the following works if you want to experiment with even fancier state-of-the-art mechanisms [1, 2].

Please write a report (report writeback refresh.pdf) that briefly summarizes <em>(i) </em>the writeback policy and refresh mechanism that you implemented, <em>(ii) </em>your observations on the performance and cache hit/miss rate compared to the simulator without realistic writeback and refresh, and <em>(iii) </em>any other optimizations you implement. Your report does not need to be more than four pages, but feel free to use more pages to present schematics, data, and graphs.

<h1>3.      Lab Resources</h1>

<h2>3.1.       Source Code</h2>

The source code that we provide for this lab is the same as the source code we provided for Lab 1. <em>You are free to choose between starting from scratch using this bare version or continuing with your simulator that includes your modifications for Lab 1 (i.e., instruction/data caches)</em>. If you decide to continue with your previous simulator, you can skip reading the rest of this section. If you decide to start from scratch, you will need to implement L1 data cache and L1 instruction cache, as described in Lab 1.

Do <strong>NOT </strong>modify any files or folders unless explicitly specified in the list below.

<ul>

 <li>Makefile</li>

 <li>run: Script that runs your simulator and compares it against the baseline simulator</li>

 <li>src/: Source code (<strong>Modifiable; feel free to add more files</strong>)

  <ul>

   <li>c: Your simulator (<strong>Modifiable</strong>)</li>

   <li>h: Your simulator (<strong>Modifiable</strong>)</li>

   <li>h: MIPS related pound defines</li>

   <li>c: Interactive shell for your simulator</li>

   <li>h: Interactive shell for your simulator</li>

  </ul></li>

 <li>inputs/: Example test inputs for your simulator (<strong>Modifiable; feel free to add more files</strong>) <strong>2. Makefile</strong></li>

</ul>

We provide a Makefile that automates the compilation and verification of your simulator.

<strong>The first time you use the </strong>Makefile <strong>you should compile the baseline simulator:</strong>

$ make basesim

This will generate basesim, which is the baseline simulator corresponding to the code we provide. You can use it to verify the output of a program you run on your simulator. Note that the output of a program should always match the output obtained by running the program on the baseline simulator. However, the execution time of a program on the two simulators will not be same after the addition of the L2 Cache and the DRAM.

To compile your simulator: $ make

To compile your simulator and check it against the baseline simulator using one or more test inputs:

$ make run INPUT=inputs/inst/addiu.x

$ make run INPUT=inputs/inst/*.x

$ make run

<h1>4.      Getting Started &amp; Tips</h1>

<h2>4.1.       Lecture Materials that Would be Useful to Study</h2>

To get started with this lab, we highly recommend you to study the following relevant materials from <em>Digital Design and Computer Architecture </em>(Spring 2020) course, which can be accessed at: <a href="https://safari.ethz.ch/digitaltechnik/spring2020/doku.php?id=schedule">https://safari.ethz.ch/digitaltechnik/spring2020/doku.php?id=schedule:</a>

<ul>

 <li>L21a: Memory Organization and Memory Technology <a href="https://safari.ethz.ch/digitaltechnik/spring2020/lib/exe/fetch.php?media=onur-digitaldesign-2020-lecture21a-memory-organization-beforelecture.pdf">PDF,</a> <a href="https://safari.ethz.ch/digitaltechnik/spring2020/lib/exe/fetch.php?media=onur-digitaldesign-2020-lecture21a-memory-organization-beforelecture.pptx">PPT,</a> <a href="https://www.youtube.com/watch?v=SdIt2seT_z0">Video</a></li>

 <li>L21b: Memory Hierarchy and Caches <a href="https://safari.ethz.ch/digitaltechnik/spring2020/lib/exe/fetch.php?media=onur-digitaldesign-2020-lecture21b-memory-hierarchy-and-caches-beforelecture.pdf">PDF,</a> <a href="https://safari.ethz.ch/digitaltechnik/spring2020/lib/exe/fetch.php?media=onur-digitaldesign-2020-lecture21b-memory-hierarchy-and-caches-beforelecture.pptx">PPT,</a> <a href="https://www.youtube.com/watch?v=mZ7CPJKzwfM">Video</a></li>

 <li>L22: More Caches <a href="https://safari.ethz.ch/digitaltechnik/spring2020/lib/exe/fetch.php?media=onur-digitaldesign-2020-lecture22-more-caches-beforelecture.pdf">PDF,</a> <a href="https://safari.ethz.ch/digitaltechnik/spring2020/lib/exe/fetch.php?media=onur-digitaldesign-2020-lecture22-more-caches-beforelecture.pptx">PPT,</a> <a href="https://www.youtube.com/watch?v=TsxQPLMXT60">Video</a></li>

 <li>L23a: Multiprocessor Caches <a href="https://safari.ethz.ch/digitaltechnik/spring2020/lib/exe/fetch.php?media=onur-digitaldesign-2020-lecture23a-multiprocessor-caches-beforelecture.pdf">PDF,</a> <a href="https://safari.ethz.ch/digitaltechnik/spring2020/lib/exe/fetch.php?media=onur-digitaldesign-2020-lecture23a-multiprocessor-caches-beforelecture.pptx">PPT,</a> <a href="https://www.youtube.com/watch?v=OUk96_Bz708">Video</a></li>

</ul>

<h2>4.2.       The Goal</h2>

<em>You can skip this section if you are already familiar with the baseline simulator we provided in Lab 1.</em>

We provide you with a skeleton of the timing simulator that models a five-stage MIPS pipeline: pipe.c and pipe.h. As it is, the simulator is already architecturally correct: it can correctly execute any arbitrary MIPS program that only uses the implemented instructions<a href="#_ftn2" name="_ftnref2"><sup>[2]</sup></a>. When the simulator detects data dependences, it correctly handles them by stalling and/or bypassing. When the simulator detects control dependences, it correctly handles them by stalling the pipeline as necessary.

By executing the following command, you can see that your simulator (sim) does indeed have identical architectural outputs (e.g., register values) as the baseline simulator (basesim) for all the test inputs that we provide in inputs/. $ make run

Your job is to model accurately the timing effects of the L2 Cache and the DRAM in your timing simulator.

<h2>4.3.       Studying the Timing Simulator</h2>

<strong>Please study </strong>pipe.c <strong>and </strong>pipe.h <strong>in detail.</strong>

The simulator models each pipeline stage as a separate function – e.g., pipe stage fetch(). The simulator models the state of the pipeline as a collection of pointers to Pipe Op structures (defined in pipe.h). Each Pipe Op represents one instruction in the pipeline by storing all of the necessary information about the instruction that is needed by the simulator. A Pipe Op structure is allocated when an instruction is fetched. It then flows through the pipeline and eventually arrives at the last stage (writeback), where it is deallocated once the instruction completes. To elaborate, each stage receives a Pipe Op from the previous stage, processes the Pipe Op, and passes it down to the next stage. The simulator models pipeline stalls by stopping the flow of Pipe Op structures and pipeline flushes by deallocating the Pipe Op structures at different stages.

<strong>5.4. Tips </strong>• <strong>Please do not distribute the provided program files. These are for exclusive individual use of each student of the Computer Architecture course. Distribution and sharing violates the copyright of the software provided to you.</strong>

<ul>

 <li><strong>Read this handout in detail.</strong></li>

 <li><strong>If needed, please ask questions to the TAs using the online Q&amp;A forum in Moodle.</strong></li>

 <li>When you encounter a technical problem, please first read the error messages. A search on the web can usually solve many debugging issues, and error messages.</li>

</ul>

<a href="#_ftnref1" name="_ftn1">[1]</a> Probing is when the tags are searched to determine the hit/miss status of an access.

<a href="#_ftnref2" name="_ftn2">[2]</a> This is not entirely true since we pose the usual restrictions on system calls, exceptions, etc., as we described in Lab 1.