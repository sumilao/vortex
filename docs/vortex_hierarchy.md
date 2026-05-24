# Vortex RTL Hierarchy

This diagram is a first-pass reading map for `hw/rtl/Vortex.sv`. It keeps the main module boundaries and data paths visible without expanding every parameterized instance.

这张图是阅读 `hw/rtl/Vortex.sv` 的第一版地图。它只保留主要模块边界和关键数据路径，不展开所有参数化实例和内部细节。

```mermaid
flowchart TB
    ext_mem["External memory interface<br/>外部内存接口<br/>mem_req_* / mem_rsp_*"]
    dcr["DCR config writes<br/>DCR 配置写入<br/>dcr_wr_valid / addr / data"]
    busy["busy<br/>忙状态"]

    subgraph vortex["Vortex"]
        direction TB

        dcr_bus["VX_dcr_bus_if<br/>DCR 配置总线<br/>broadcast to clusters"]

        subgraph l3["System memory level<br/>系统内存层"]
            per_cluster_bus["per_cluster_mem_bus_if<br/>每个 cluster 的内存总线"]
            l3cache["VX_cache_wrap<br/>L3 cache / passthrough<br/>L3 缓存或直通"]
            mem_bus["mem_bus_if<br/>对外内存总线"]
        end

        subgraph clusters["g_clusters[NUM_CLUSTERS]"]
            direction TB

            subgraph cluster["VX_cluster"]
                direction TB
                cluster_dcr["cluster_dcr_bus_if<br/>cluster 级 DCR 总线"]
                gbar_cluster["VX_gbar_arb + VX_gbar_unit<br/>全局 barrier 仲裁/执行<br/>if GBAR_ENABLE"]
                l2cache["VX_cache_wrap<br/>L2 cache / passthrough<br/>L2 缓存或直通"]
                per_socket_bus["per_socket_mem_bus_if<br/>每个 socket 的内存总线"]

                subgraph sockets["g_sockets[NUM_SOCKETS]"]
                    direction TB

                    subgraph socket["VX_socket"]
                        direction TB
                        socket_dcr["socket_dcr_bus_if<br/>socket 级 DCR 总线"]
                        gbar_socket["VX_gbar_arb<br/>socket 内 barrier 仲裁<br/>if GBAR_ENABLE"]

                        icache["VX_cache_cluster<br/>I-cache<br/>指令缓存"]
                        dcache["VX_cache_cluster<br/>D-cache<br/>数据缓存"]
                        l1arb["VX_mem_arb<br/>L1 内存仲裁<br/>prioritizes I-cache on port 0"]

                        subgraph cores["g_cores[SOCKET_SIZE]"]
                            direction TB

                            subgraph core["VX_core"]
                                direction TB
                                dcr_data["VX_dcr_data<br/>DCR 配置寄存器"]
                                schedule["VX_schedule<br/>warp 调度"]
                                fetch["VX_fetch<br/>取指"]
                                decode["VX_decode<br/>译码"]
                                issue["VX_issue<br/>发射"]
                                execute["VX_execute<br/>执行"]
                                commit["VX_commit<br/>提交/写回控制"]
                                mem_unit["VX_mem_unit<br/>LSU 到 D-cache 的转换"]

                                schedule --> fetch --> decode --> issue --> execute --> commit
                                commit -->|"writeback_if 写回"| issue
                                schedule <-->|"warp/branch/commit sched<br/>warp/分支/提交调度反馈"| execute
                                dcr_data -->|"base_dcrs 基础配置"| schedule
                                dcr_data -->|"base_dcrs 基础配置"| execute
                                execute -->|"lsu_mem_if LSU 访存接口"| mem_unit
                            end
                        end

                        fetch -->|"icache_bus_if 指令访问"| icache
                        mem_unit -->|"dcache_bus_if 数据访问"| dcache
                        icache --> l1arb
                        dcache --> l1arb
                    end
                end

                l1arb --> per_socket_bus --> l2cache
            end
        end

        l2cache --> per_cluster_bus --> l3cache --> mem_bus
        dcr_bus --> cluster_dcr --> socket_dcr --> dcr_data
    end

    dcr --> dcr_bus
    mem_bus --> ext_mem
    clusters --> busy
```

## Suggested Reading Order

建议按这个顺序阅读源码：

1. `hw/rtl/Vortex.sv`: top-level memory ports, DCR bus, L3, cluster generation.
2. `hw/rtl/VX_cluster.sv`: L2 cache and socket generation.
3. `hw/rtl/VX_socket.sv`: I-cache, D-cache, L1 arbitration, core generation.
4. `hw/rtl/core/VX_core.sv`: pipeline stages and the LSU memory path.
5. `hw/rtl/core/VX_execute.sv`: ALU/LSU/FPU/TCU/SFU dispatch.
6. `hw/rtl/core/VX_mem_unit.sv`: LSU requests into the D-cache bus.

中文对照：

1. `hw/rtl/Vortex.sv`：顶层内存端口、DCR 配置总线、L3 缓存、cluster 生成。
2. `hw/rtl/VX_cluster.sv`：L2 缓存和 socket 生成。
3. `hw/rtl/VX_socket.sv`：I-cache、D-cache、L1 内存仲裁、core 生成。
4. `hw/rtl/core/VX_core.sv`：核心流水线阶段，以及 LSU 访存路径。
5. `hw/rtl/core/VX_execute.sv`：ALU、LSU、FPU、TCU、SFU 的执行单元分发。
6. `hw/rtl/core/VX_mem_unit.sv`：LSU 请求如何转换并送到 D-cache 总线。

## Terms

| Term | 中文 | Meaning |
| --- | --- | --- |
| DCR | 设备控制寄存器 | Host or control path writes configuration/state into the device. |
| I-cache | 指令缓存 | Fetch stage reads instructions through this path. |
| D-cache | 数据缓存 | LSU load/store requests go through this path. |
| LSU | 加载/存储单元 | Handles memory operations from execute stage. |
| GBAR | 全局 barrier | Synchronization path enabled by `GBAR_ENABLE`. |
| passthrough | 直通 | Cache wrapper can bypass cache behavior when that cache level is disabled. |
