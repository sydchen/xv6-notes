推薦一門 MIT EECS 的神課：**6.1810: Operating System Engineering**。

這門課最迷人的地方，是它不是只講理論，而是直接帶你動手改一個 Unix-like 作業系統 —— **xv6**。課程會一步一步帶你理解：

* virtual memory 是怎麼被實作出來的
* file system 在 kernel 裡到底長什麼樣子
* threads / context switch 背後的機制
* interrupt、system call 如何在 user 與 kernel 之間切換
* mmap 怎麼和記憶體管理整合
* 甚至還有機會寫一個非常陽春的 Intel e1000 network driver

很多平常在 Linux 上理所當然的東西，實際拆開來看之後，會發現每一層都很精妙。

其實不一定要正式修課，直接把課程 PDF 打開，照著進度做 homework 和 labs，一步一步實作下來，就能慢慢掌握 OS kernel 的核心技術。這門課實作非常多，如果你是那種喜歡看 code、改 code、追 stack trace、debug 到凌晨的人，應該會很喜歡。

課程不太輕鬆。大概需要花幾個月慢慢吸收，但那種把教科書上的內容，直接實作出來的感覺真的很值得。

---

## English Version

I really want to recommend an amazing MIT EECS course: **6.1810: Operating System Engineering**.

What makes this course so compelling is that it does not stop at theory. You directly work on a Unix-like operating system called **xv6**. Step by step, the course helps you understand:

* how virtual memory is actually implemented
* what a file system really looks like inside the kernel
* the mechanisms behind threads and context switching
* how interrupts and system calls switch between user mode and kernel mode
* how `mmap` integrates with memory management
* and even how to write a very minimal Intel e1000 network driver

Many things that feel "obvious" when using Linux become much more interesting once you take them apart layer by layer.

You do not necessarily need to enroll formally. You can open the course PDFs, follow the schedule, and work through the homework and labs. By implementing each piece yourself, you gradually build a solid understanding of core OS kernel techniques. This course is highly hands-on, so if you enjoy reading code, modifying code, chasing stack traces, and debugging late into the night, you will probably love it.

To be honest, the course is not easy. It usually takes a few months to absorb everything, but the experience of implementing textbook concepts yourself is absolutely worth it.
