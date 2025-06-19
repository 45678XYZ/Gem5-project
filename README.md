# Gem5 變更項目
### Q2
在 `gem5/configs/common/Caches.py` 後加上 L3cache
```python
class L3Cache(Cache):
    assoc = 64
    tag_latency = 32
    data_latency = 32
    response_latency = 20
    mshrs = 32
    tgts_per_mshr = 24
    write_buffers = 16
```

在 `gem5/configs/common/CacheConfig.py` 修改 config_cache，加上 L3cache
```python
    if options.cpu_type == "O3_ARM_v7a_3":
        try:
            from cores.arm.O3_ARM_v7a import *
        except:
            print("O3_ARM_v7a_3 is unavailable. Did you compile the O3 model?")
            sys.exit(1)

        dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = \
            O3_ARM_v7a_DCache, O3_ARM_v7a_ICache, O3_ARM_v7aL2, \
            O3_ARM_v7aWalkCache, O3_ARM_v7aL3

    else:
        dcache_class, icache_class, l2_cache_class, walk_cache_class, l3_cache_class = \
            L1_DCache, L1_ICache, L2Cache, None, L3Cache

        if buildEnv['TARGET_ISA'] == 'x86':
            walk_cache_class = PageTableWalkerCache
```

```python
    if options.l2cache and options.l3cache:
        # Provide a clock for the L2, L3 and the L1-to-L2, L2-to-L3 bus here as they
        # are not connected using addTwoLevelCacheHierarchy. Use the
        # same clock as the CPUs.
        system.l2 = l2_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l2_size,
                                   assoc=options.l2_assoc)
	system.l3 = l3_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l3_size,
                                   assoc=options.l3_assoc)

        system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
	system.tol3bus = L3XBar(clk_domain = system.cpu_clk_domain)

        system.l2.cpu_side = system.tol2bus.master
	system.l3.cpu_side = system.tol3bus.master

        system.l2.mem_side = system.tol3bus.slave
	system.l3.mem_side = system.membus.slave

    elif options.l2cache:
        # Provide a clock for the L2 and the L1-to-L2 bus here as they
        # are not connected using addTwoLevelCacheHierarchy. Use the
        # same clock as the CPUs.
        system.l2 = l2_cache_class(clk_domain=system.cpu_clk_domain,
                                   size=options.l2_size,
                                   assoc=options.l2_assoc)

        system.tol2bus = L2XBar(clk_domain = system.cpu_clk_domain)
        system.l2.cpu_side = system.tol2bus.master
        system.l2.mem_side = system.membus.slave
```

在 `gem5/src/mem/XBar.py` 加入 L3XBar
```python
class L3XBar(CoherentXBar):
    width = 32
    frontend_latency = 1
    forward_latency = 0
    response_latency = 1
    snoop_response_latency = 1
    snoop_filter = SnoopFilter(lookup_latency = 0)
    point_of_unification = True
```

在 `gem5/src/cpu/BaseCPU.py` 引入 L3XBar 並新增 L3cache 的 function
```python
from XBar import L3XBar
```

```python
    def addThreeLevelCacheHierarchy(self, ic, dc, l3c, iwc=None, dwc=None,
                                  xbar=None):
        self.addPrivateSplitL1Caches(ic, dc, iwc, dwc)
        self.toL3Bus = L3XBar()
        self.connectCachedPorts(self.toL3Bus)
        self.l3cache = l3c
        self.toL2Bus.master = self.l3cache.cpu_side
        self._cached_ports = ['l3cache.mem_side']
```

在 `gem5/configs/common/Options.py` 的 addNoISAOptions 裡新增啟用 L3cache 的參數
```python
    parser.add_option("--l3cache", action="store_true")
```

執行
```
./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

### Q3
**2-way**
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

**full-way**
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=16384 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

### Q4
**Original replacement policy**  
Code 和原本一樣，執行
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

**Frequency based replacement policy**  
在 `gem5/configs/common/Caches.py` 加上 replacement policy
```python
class L3Cache(Cache):
    assoc = 64
    tag_latency = 32
    data_latency = 32
    response_latency = 20
    mshrs = 32
    tgts_per_mshr = 24
    write_buffers = 16
    replacement_policy = Param.BaseReplacementPolicy(LFURP(),"Replacement policy")
```
執行
```
./build/X86/gem5.opt configs/example/se.py -c ./quicksort --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=2 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

### Q5
**Write Back**  
Code 和原本一樣，執行
```
./build/X86/gem5.opt configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=4 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```

**Write Through**  
在 `gem5/src/mem/cache/base.cc` 的 BaseCache 中加入
```python
        if (blk->isWritable()) {
            PacketPtr writeclean_pkt = writecleanBlk(blk, pkt->req->getDest(), pkt->id);
            writebacks.push_back(writeclean_pkt);
        }
```

執行
```
./build/X86/gem5.opt configs/example/se.py -c ./multiply --cpu-type=TimingSimpleCPU --caches --l2cache --l3cache --l3_assoc=4 --l1i_size=32kB --l1d_size=32kB --l2_size=128kB --l3_size=1MB --mem-type=NVMainMemory --nvmain-config=../NVmain/Config/PCM_ISSCC_2012_4GB.config
```
