# 主要 CPU 微架构列表（List of the Major CPU Microarchitectures）

下表列出了 Intel、AMD 和基于 ARM 的厂商最新的 ISA 及微架构。当然，并非所有设计都包含在内，本表仅收录书中引用的，或代表该平台演进中重大转变的微架构。

-----------------------------------------------------------------
    Name         Three-letter     Year released     Supported ISA
                  acronym                          client/server
                                                       chips
--------------  ---------------  ---------------  ---------------
   Nehalem           NHM              2008             SSE4.2

Sandy Bridge         SNB              2011              AVX

   Haswell           HSW              2013              AVX2

   Skylake           SKL              2015         AVX2 / AVX512

 Sunny Cove          SNC              2019             AVX512

 Golden Cove         GLC              2021         AVX2 / AVX512 

 Redwood Cove        RWC              2023         AVX2 / AVX512 

  Lion Cove          LNC              2024             AVX2

-----------------------------------------------------------------

Table: 近期 Intel Core 微架构列表。

----------------------------------------------
    Name       Year released    Supported ISA
------------  ---------------  ---------------
 Steamroller       2014              AVX

  Excavator        2015              AVX2

   Zen             2017              AVX2

   Zen2            2019              AVX2

   Zen3            2020              AVX2

   Zen4            2022             AVX512

   Zen5            2024             AVX512

----------------------------------------------

Table: 近期 AMD 微架构列表。


------------------------------------------------------------------
    ISA        Year of ISA      Arm uarchs         Third-party
                 release         (latest)            uarchs
------------  ---------------  --------------   ------------------
  ARMv8-A          2011          Cortex-A73        Apple A7-A10;
                                                  Qualcomm Kryo;
                                                 Samsung M1/M2/M3

 ARMv8.2-A         2016         Neoverse N1;         Apple A11;
                                 Cortex-X1           Samsung M4;
                                                    Ampere Altra

 ARMv8.4-A         2017         Neoverse V1        AWS Graviton3;
                                                   Apple A13, M1

 ARMv9.0-A         2018         Neoverse N2;    Microsoft Cobalt 100;
(64bit-only)                    Neoverse V2;        NVIDIA Grace;
                                 Cortex X3          AWS Graviton4;

 ARMv8.6-A         2019             ---          Apple A15, A16, M2, M3
(64bit-only)

 ARMv9.2-A         2020          Cortex X4             Apple M4
------------------------------------------------------------------

Table: 近期 ARM ISA 列表及其自研与第三方实现。

\bibliography{biblio}
