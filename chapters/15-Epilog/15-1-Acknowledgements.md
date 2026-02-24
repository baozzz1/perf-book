# Acknowledgments

I write this section with profound gratitude to all the people mentioned below. I feel very humble to deliver to you all the knowledge these people have.

   <br> 

\setlength\intextsep{0pt}

![](../../img/contributors/MarkDawson_circle.png) 
Mark E. Dawson, Jr. authored [LowLatency] "Low Latency Tuning Techniques" and [ContinuousProfiling] "Continuous Profiling". He has also contributed a lot to the first edition of the book. Mark is a recognized expert in the High-Frequency Trading industry, currently working at WH Trading. Mark also has a blog ([https://www.jabperf.com](https://www.jabperf.com)) where he writes about low latency and other performance optimizations.

   <br> 

![](../../img/contributors/UniversityZaragoza_circle.png) 
Agustín Navarro-Torres, Jesús Alastruey-Benedé, Pablo Ibáñez-Marín, Víctor Viñals-Yúfera from the University of Zaragoza authored [Sensitivity2LLC] "Case Study: Sensitivity to Last Level Cache Size". They are researchers in the field of computer science and have published multiple papers on topics related to performance engineering.

   <br> 

![](../../img/contributors/JanWassenberg_circle.png) 
Jan Wassenberg authored [secIntrinsicLibraries] "Wrapper Libraries for Intrinsics" and also proposed many improvements to [Vectorization] "Vectorization" and [SIMD] "SIMD Multiprocessors". Jan is a software engineer at Google DeepMind, where he leads the development of Gemma.cpp.[^1] He also has authored
multiple research papers. His personal webpage is [https://wassenberg.dreamhosters.com](https://wassenberg.dreamhosters.com).

   <br> 

![](../../img/contributors/SwarupSahoo_circle.png) 
Swarup Sahoo helped me with writing about AMD PMU features in [PmuChapter], and authored [IntelVtuneOverview] about AMD uProf. Swarup is a senior developer at AMD, where he works on the uProf performance analysis tool for HPC (OpenMP, MPI) applications. Swarup's LinkedIn page can be found at [https://www.linkedin.com/in/swarupsahoo](https://www.linkedin.com/in/swarupsahoo).

   <br> 

![](../../img/contributors/AloisKraus_circle.png) 
Alois Kraus authored [ETW] "Event Tracing for Windows", and Appendix D. He developed `ETWAnalyzer`, a command-line tool for analyzing ETW files with simple queries. He is employed by Siemens Healthineers where he studies the performance of large software systems. Alois' personal webpage and blog: [https://aloiskraus.wordpress.com](https://aloiskraus.wordpress.com).

   <br> 

![](../../img/contributors/MarcoCastorina_circle.png) 
Marco Castorina authored [Tracy] "Specialized and Hybrid Profilers" which showcases performance profiling with Tracy. Marco currently works on the games graphics performance team at AMD, focusing on DX12. Also, he is the co-author of a book titled "Mastering Graphics Programming with Vulkan". Marco's personal web page is [https://marcocastorina.com](https://marcocastorina.com).

   <br> 

![](../../img/contributors/LallySingh_circle.png) 
Lally Singh has authored [MarkerAPI] about Marker APIs. Lally is currently at Tesla, his prior work includes Datadog's performance team, Google's Search performance team, low-latency trading systems, and embedded real-time control systems. Lally has a PhD in CS from Virginia Tech, focusing on scalability in distributed VR.

   <br> 

![](../../img/contributors/DickSites_circle.png) 
Richard L. Sites provided a technical review of the book. He is a veteran of the semiconductor industry and has spent most of his career at the boundary between hardware and software, particularly in CPU/software performance interactions. Richard invented the performance counters found in most modern processors. He had worked at DEC, Adobe, Google, and Tesla. His personal page is [https://sites.google.com/site/dicksites](https://sites.google.com/site/dicksites).

   <br> 

![](../../img/contributors/MattGodbolt_circle.png) 
Matt Godbolt provided a technical review of the book. Matt is a creator of Compiler Explorer, an extremely popular tool among software developers. He is a C++ developer passionate about high-performance code. Matt has over twenty years of professional experience in computer game programming, systems design, real-time embedded systems, and high-frequency trading. He is also a speaker, a blogger, and a podcaster. His personal blog is [https://xania.org](https://xania.org).

  <br> 

Also, I would like to thank the following people. Jumana Mundichipparakkal from Arm, for helping me write about Arm PMU features in [PmuChapter]. Yann Collet, the author of Zstandard, for providing me with the information about the internal workings of Zstd for [ThreadCountScalingStudy]. Ciaran McHale, for finding tons of grammar mistakes in my initial draft. Nick Black for proofreading and editing the final version of the book. Peter Veentjer, Amir Aupov, and Charles-Francois Natali for various edits and suggestions.

I'm also thankful to the whole performance community for countless blog articles and papers. I was able to learn a lot from reading blogs by Travis Downs, Daniel Lemire, Andi Kleen, Agner Fog, Bruce Dawson, Brendan Gregg, and many others. I stand on the shoulders of giants, and the success of this book should not be attributed only to myself. This book is my way to thank and give back to the whole community.

A special "thank you" goes to my family, who were patient enough to tolerate me missing weekend trips and evening walks. Without their support, I wouldn't have finished this book.

Images were created with excalidraw.com. Cover design by Darya Antonova. The fonts used on the cover of this book are Bebas Neue and Raleway, both provided under the Open Font License. Bebas Neue was designed by Ryoichi Tsunekawa. Raleway was designed by Matt McInerney, Pablo Impallari, and Rodrigo Fuenzalida.

I should also mention contributors to the first edition of this book. Below I only list names and section titles. More detailed acknowledgments are available in the first edition.

* Mark E. Dawson, Jr. wrote sections [secDTLB] "Optimizing For DTLB", [FeTLB] "Optimizing for ITLB", [CacheWarm] "Cache Warming", [SysTune] "System Tuning", and a few other.
* Sridhar Lakshmanamurthy, authored a large part of [uarch] about CPU microarchitecture.
* Nadav Rotem helped write [Vectorization] about vectorization.
* Clément Grégoire authored [ISPC] about ISPC compiler.
* Reviewers: Dick Sites, Wojciech Muła, Thomas Dullien, Matt Fleming, Daniel Lemire, Ahmad Yasin, Michele Adduci, Clément Grégoire, Arun S. Kumar, Surya Narayanan, Alex Blewitt, Nadav Rotem, Alexander Yermolovich, Suchakrapani Datt Sharma, Renat Idrisov, Sean Heelan, Jumana Mundichipparakkal, Todd Lipcon, Rajiv Chauhan, Shay Morag, and others.

[^1]: Gemma.cpp, LLM inference on CPU - [https://github.com/google/gemma.cpp](https://github.com/google/gemma.cpp)
