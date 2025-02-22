注：经chatgpt和deepseek翻译自论文 “Scheduling Algorithms for Multiprogramming in a Hard-Real-Time Environment”，略作整理。原文链接：https://dl.acm.org/doi/abs/10.1145/321738.321743

# 硬实时环境中的多任务调度算法

**C. L. Liu**  
麻省理工学院项目 MAC  

**James W. Layland**  
加州理工学院喷气推进实验室  

---

## 摘要  
本文从需要保证服务的程序功能特性出发，研究了单处理器上的多任务调度问题。研究表明，最优的固定优先级调度器在处理器利用率方面存在一个上限，该上限对于较大的任务集可能低至 70%。然而，基于当前截止期动态地分配优先级，可以实现处理器的完全利用。此外，本文还探讨了这两种调度技术的结合。

**关键词和短语：** 实时多任务、调度、多任务调度、动态调度、优先级分配、处理器利用率、基于截止期的调度  
**CR 分类：** 3.80, 3.82, 3.83, 4.32  

---

## 1. 引言  
近年来，计算机在工业过程的控制和监测中的应用大幅扩展，并且未来可能更加显著。在这种应用中，计算机通常会被同时用于一定数量的时间敏感控制和监测功能，以及非时间敏感的批处理任务流。然而，在某些场景中，系统中不存在非时间敏感的任务，只有通过对时间敏感的控制和监测功能进行仔细调度，才能有效利用计算机。这类系统可以称为“纯过程控制”系统，为本文的组合调度分析提供了背景。

本文研究了两种适用于此类编程的调度算法，这两种算法都基于优先级驱动并支持抢占，也就是说，任何较高优先级任务的请求都会中断当前任务的处理。第一种算法采用固定优先级分配，处理器利用率可达到约 70% 或更高。第二种算法通过动态优先级分配可以实现处理器的完全利用。本文还讨论了这两种算法的结合。

---

## 2. 背景  
一个过程控制计算机可以执行一个或多个控制和监测功能。例如，为跟踪轨道中的航天器而调整天线的指向就是这样的功能之一。每个需要执行的功能都与一个或多个任务集相关联。这些任务中有些是为响应由计算机控制或监测的设备事件，其他任务则是为响应其他任务的事件。在请求事件发生之前，响应任务不能开始；在请求事件发生后，一定时间内任务必须完成。保证在该时间范围内完成服务的环境被分类为“硬实时”，与此相反的是“软实时”环境，其对响应时间的统计分布是可以接受的。

现有关于多任务的文献大多涉及商业分时系统的统计分析（文献 [2] 包含了广泛的相关书目）。另一部分文献讨论了在多处理器配置下批处理或分时混合批处理的调度问题 [3-8]。仅有少数论文直接研究了“硬实时”的编程问题。例如，Manacher [1] 提出了一个用于硬实时环境中任务调度生成的算法，尽管考虑了多个截止期，但其结果仅限于所有任务共享相同请求时间的非现实场景。Lampson [9] 总体上探讨了软件调度问题，并提出了一组 ALGOL 多任务编程过程，这些过程可以作为软件实现或嵌入在特殊用途调度器中。

Martin [10] 的著作描述了被视为“实时”系统的各种类型，并有条理地讨论了编程这些系统时遇到的问题。Jirauch [11] 的论文中提到的自动检查系统软件也强调了对实时软件开发需要保持严格的工程管理控制。这些讨论凸显了当前系统设计方法的局限性，表明需要一种更系统化的软件设计方法。

---

## 3. 环境假设  
为了对硬实时环境中的程序行为获得分析性结果，需要对该环境做出一些假设。这些假设并非都绝对必要，放宽这些假设的影响将在后续部分讨论。

- **(A1)** 对于所有具有硬截止期的任务，其请求是周期性的，并且请求之间的时间间隔是恒定的；  
- **(A2)** 截止期仅由可运行性限制组成，即每个任务必须在下一次请求发生之前完成；  
- **(A3)** 任务是独立的，即某任务的请求不依赖于其他任务的启动或完成；  
- **(A4)** 每个任务的运行时间是恒定的，并且不随时间变化。运行时间是指处理器在不被中断的情况下执行该任务所需的时间；  
- **(A5)** 系统中的非周期性任务是特殊的，例如初始化或故障恢复程序；它们会在运行时取代周期性任务，但它们本身没有严格的关键截止期。

假设 (A1) 与 Martin [2] 的观点相对立，但对于纯过程控制场景是有效的。假设 (A2) 消除了单个任务的排队问题。为使假设 (A2) 成立，每个外设功能可能需要一小部分但关键的缓冲硬件。假设 (A3) 不排除以下情况：任务 $` \tau_r `$ 的出现次数必须是任务 $` \tau_i `$ 出现次数的固定倍数。例如，这种情况可以通过选择任务 $` \tau_r `$ 和 $` \tau_i `$ 的周期，使得 $` \tau_r `$ 的周期是 $` \tau_i `$ 的 $` N `$ 倍，从而 $` \tau_i `$ 的第 $` N `$ 次请求与 $` \tau_r `$ 的第一次请求重合来建模。

假设 (A4) 中的运行时间可以解释为任务的最大处理时间。通过这种方式，请求后续任务的管理时间和抢占的开销也可被考虑在内。现代计算机系统中，由于主存储器和辅助存储器之间的大容量传输以及程序执行的重叠，假设 (A4) 应该是一个良好的近似，即使它并不完全准确。

这些假设允许用**请求周期**和**运行时间**这两个参数来完全描述一个任务。除非另有说明，本文将使用 $` \tau_1, \tau_2, \dots, \tau_m `$ 表示 $` m `$ 个周期性任务，其请求周期分别为 $` T_1, T_2, \dots, T_m `$，运行时间分别为 $` C_1, C_2, \dots, C_m `$。任务的请求率定义为其请求周期的倒数。

调度算法是一组规则，用于确定某一时刻要执行的任务。本文研究的调度算法是抢占式和基于优先级驱动的。这意味着，当有更高优先级任务的请求时，会立即中断当前任务并启动新请求的任务。因此，此类算法的规范化相当于任务优先级分配方法的规范化。如果任务的优先级被一次性分配且不变，则该算法被称为静态调度算法，也称为固定优先级调度算法。如果任务的优先级可能随每次请求而变化，则称为动态调度算法。如果部分任务的优先级是固定的，而其余任务的优先级随着每次请求而变化，则该算法称为混合调度算法。

---

## 4. 固定优先级调度算法  
在本节中，我们推导出一种优先级分配规则，该规则可实现最优的静态调度算法。决定这一规则的重要概念是任务的“关键时刻”。

任务请求的截止期被定义为该任务下一次请求的时间。对于按照某种调度算法调度的一组任务，如果在某一时刻未能完成的请求达到其截止期，则称该时刻发生了“溢出”。对于给定的一组任务，如果调度算法能够使所有任务都能按其截止期完成，则称该算法是“可行的”。我们将某一任务的请求的响应时间定义为该请求的时间跨度，即从发出请求到完成响应的时间。

任务的关键时刻被定义为该任务请求响应时间最长的时刻。关键时间区间指从关键时刻到完成该任务对应请求响应的时间区间。我们有如下定理：

**定理 1：** 对于任何任务，当该任务与所有更高优先级任务的请求同时发生时，便出现该任务的关键时刻。

**证明：** 设 $` \tau_1, \tau_2, \dots, \tau_m `$ 表示一组按照优先级排列的任务，其中 $` \tau_m `$ 是优先级最低的任务。考虑在时刻 $` t_1 `$ 对 $` \tau_m `$ 发出的一个请求。假设在 $` t_1 `$ 和 $` t_1 + T_m `$ 之间，即 $` \tau_m `$ 的下一次请求的时刻，任务 $` \tau_1, \tau_2, \dots, \tau_{m-1} `$ 的请求分别在时刻 $` t_2, t_2 + T_1, t_2 + 2T_1, \dots `$ 发生，如图 1 所示。显然，任务 $` \tau_1 `$ 对 $` \tau_m `$ 的抢占将导致任务 $` \tau_m `$ 在 $` t_1 `$ 时刻发出的请求的完成时间出现一定延迟，除非任务 $` \tau_m `$ 在 $` t_2 `$ 前完成。此外，从图 1 可以立即看出，提前 $` t_2 `$ 的请求时间并不会加速 $` \tau_m `$ 的完成时间。提前请求的完成时间要么不变，要么被延迟。因此，当 $` t_1 `$ 与 $` t_2 `$ 重合时，$` \tau_m `$ 的完成延迟最大。对所有 $` \tau_1, \tau_2, \dots, \tau_{m-1} `$ 任务重复此论证，我们就证明了该定理。

此结果的一个重要价值在于，可以通过简单的直接计算来判断给定的优先级分配是否会产生可行的调度算法。具体来说，如果所有任务在其关键时刻的请求都能在截止期之前完成，则调度算法是可行的。举个例子，考虑一组包含两个任务 $` \tau_1 `$ 和 $` \tau_2 `$，它们的周期分别为 $` T_1 = 2 `$，$` T_2 = 5 `$，运行时间分别为 $` C_1 = 1 `$，$` C_2 = 1 `$。如果我们让 $` \tau_1 `$ 为高优先级任务，那么从图 2（a）可以看到这种优先级分配是可行的。此外，$` C_2 `$ 的值最多可以增加到 2，但不能再增加，如图 2（b）所示。另一方面，如果我们让 $` \tau_2 `$ 为高优先级任务，那么 $` C_1 `$ 和 $` C_2 `$ 的值都不能超过 1，如图 2（c）所示。

<div align="center">
  <img src="https://github.com/user-attachments/assets/eec76d83-79a5-4ada-925b-60abf68ebc19" width="600" />
</div>

<div align="center">
  <img src="https://github.com/user-attachments/assets/4c2375bc-bba3-4c0f-b636-82a551215d4c" width="600" />
</div>

**定理1** 中的结果还暗示了一种最优的优先级分配方法，正如 **定理2** 中将要陈述的那样。为了阐明这一普遍性结果，考虑调度两个任务 $` \tau_1 `$ 和 $` \tau_2 `$ 的情况。设 $` T_1 `$ 和 $` T_2 `$ 分别为这两个任务的请求周期，且 $` T_1 < T_2 `$。如果我们让 $` \tau_1 `$ 为高优先级任务，那么，根据定理1，必须满足以下不等式：
(1)

$$
\left\lfloor \frac{T_2}{T_1} \right\rfloor C_1 + C_2 \leq T_2
$$

如果我们让 $` \tau_2 `$ 为高优先级任务，那么必须满足以下不等式：
(2)

$$
C_1 + C_2 \leq T_1
$$

因此：

$$
\left\lfloor \frac{T_1}{T_2} \right\rfloor C_1 + \left\lfloor \frac{T_2}{T_1} \right\rfloor C_2 \leq \left\lfloor \frac{T_2}{T_1} \right\rfloor T_2 \leq T_2
$$

由(2) 可推导出 (1)。即，当 $` T_1 < T_2 `$ 且 $` C_1、 C_2 `$ 在 $` \tau_1 `$ 的优先级高于 $` \tau_2 `$ 时，任务调度是可行的。那么当 $` \tau_2 `$ 的优先级高于 $` \tau_1 `$ 时也是可行的，但反之不成立。因此，我们可以将更高的优先级分配给 $` \tau_1 `$，将较低的优先级分配给 $` \tau_2 `$。

因此，一个“可行的”的优先级分配规则是根据任务的请求速率来分配优先级，而与其运行时间无关。具体来说，请求速率较高的任务将拥有较高的优先级。这样的优先级分配方法将被称为速率单调优先级分配。事实证明，这种优先级分配是最优的，因为没有其他的固定优先级分配规则可以调度**速率单调优先级分配规则**无法调度的任务集。

- 注：条件 (1) 是优先级分配可行的必要条件，但并非充分条件。  
- $` \lfloor x \rfloor `$ 表示小于或等于 $` x `$ 的最大整数（向下取整）。  
- $` \lceil x \rceil `$ 表示大于或等于 $` x `$ 的最小整数（向上取整）。  

**定理 2：** 如果某个任务集存在可行的优先级分配，则速率单调优先级分配对该任务集也是可行的。

**证明：** 设 $` \tau_1, \tau_2, \dots, \tau_m `$ 为一组 $` m `$ 个任务，且其优先级分配是可行的。设 $` \tau_1 `$ 和 $` \tau_2 `$ 为在这种分配中具有相邻优先级的两个任务，其中 $` \tau_1 `$ 的优先级高于 $` \tau_2 `$。假设 $` T_1 > T_2 `$。我们交换 $` \tau_1 `$ 和 $` \tau_2 `$ 的优先级。显然，交换后的优先级分配仍然是可行的。由于速率单调优先级分配可以从任何优先级顺序中通过一系列的成对交换优先级（如上所述）得到，因此该定理得证。

---

## 5. 可实现的处理器利用率  
此时，已有工具可以确定固定优先级系统中处理器利用率的最小上界。我们将（处理器）**利用率因子**定义为在执行任务集过程中所占用的处理器时间的比例。即，利用率因子等于 1 减去空闲处理器时间的比例。由于 $` \frac{C_j}{T_i} `$ 是执行任务 $` \tau_i `$ 时所占用的处理器时间比例，因此，对于 $` m `$ 个任务，利用率因子为：

$$
U = \sum_{i=1}^{m} \frac{C_i}{T_i}
$$

虽然通过增大 $` C_i `$ 或减小 $` T_i `$ 可以提高处理器利用率因子，但它的上限受到所有任务在其关键时刻满足截止期要求的限制。显然，讨论处理器利用率因子可以有多小并没有太大意义。但是，讨论处理器利用率因子可以有多大则是有意义的，因此我们明确以下3个概念。

1. 对于某一优先级分配，如果一组任务的优先级分配对于这组任务是可行的，并且如果增加任何任务的运行时间会导致该优先级分配变得不可行，那么我们称这组任务完全利用了处理器；  
2. 对于给定的固定优先级调度算法，利用率因子的最小上界是所有完全利用处理器的任务集的利用率因子的最小值；  
3. 对于所有处理器利用率因子低于这个上界的任务集，都存在一个可行的固定优先级分配。另一方面，只有当任务的 $` T_i `$ 之间有适当的关系时，才能实现超过此上界的利用率。

由于速率单调优先级分配是最优的，速率单调优先级分配对于给定任务集所实现的利用率因子大于或等于该任务集的任何其他优先级分配所实现的利用率因子。**因此，要确定的最小上界是所有可能的请求周期和任务运行时间下，速率单调优先级分配对应的利用率因子的下确界**。这个上界首先是针对两个任务确定的，然后再扩展到任意数量的任务。

**定理 3：** 对于一组具有固定优先级分配的两个任务，处理器利用率因子的最小上界为 $` U = 2 \left( 2^{\frac{1}{2}} - 1 \right) `$。

**证明：** 设 $` \tau_1 `$ 和 $` \tau_2 `$ 为两个任务，它们的周期分别为 $` T_1 `$ 和 $` T_2 `$，运行时间分别为 $` C_1 `$ 和 $` C_2 `$。假设 $` T_2 > T_1 `$。根据速率单调优先级分配，$` \tau_1 `$ 的优先级高于 $` \tau_2 `$。在 $` \tau_2 `$ 的关键时间区间内，有 $` \left\lfloor \frac{T_2}{T_1} \right\rfloor `$ 次 $` \tau_1 `$ 的请求。现在，我们调整 $` C_2 `$ 以完全利用可用的处理器时间，位于关键时间区间内。此时会有两种情况：

**情况 1：** 运行时间 $` C_1 `$ 足够短，以至于 $` \tau_1 `$ 在 $` \tau_2 `$ 的关键时间区间内的所有请求都在第二次 $` \tau_2 `$ 请求之前完成。即：

$$
C_1 \leq T_2 - T_1 \left\lfloor \frac{T_2}{T_1} \right\rfloor
$$

因此，$` C_2 `$ 的最大可能值为：

$$
C_2 = T_2 - C_1 \left\lfloor \frac{T_2}{T_1} \right\rfloor
$$

相应的处理器利用率因子为：

$$
U = 1 + C_1 \left[ \frac{1}{T_1} - \frac{1}{T_2} \left\lfloor \frac{T_2}{T_1} \right\rfloor \right]
$$

在这种情况下，处理器利用率因子 $` U `$ 随着 $` C_1 `$ 的增加而单调递减。

**情况 2：** 任务 $` r_1 `$ 的第 $` \left\lfloor T_2 / T_1 \right\rfloor `$ 次请求的执行与任务 $` r_2 `$ 的第二次请求发生重叠。在这种情况下，有：

$$
C_1 \geq T_2 - T_1 \left\lfloor \frac{T_2}{T_1} \right\rfloor
$$

由此可得，$` C_2 `$ 的最大可能值为：

$$
C_2 = -C_1 \left\lfloor \frac{T_2}{T_1} \right\rfloor + T_1 \left\lfloor \frac{T_2}{T_1} \right\rfloor
$$

相应的处理器利用率因子为：

$$
U = \frac{T_1}{T_2} \left\lfloor \frac{T_2}{T_1} \right\rfloor + C_1 \left[ \frac{1}{T_1} - \frac{1}{T_2} \left\lfloor \frac{T_2}{T_1} \right\rfloor \right]
$$

在这种情况下，处理器利用率 $` U `$ 随 $` C_1 `$ 增加单调递增。

两种情况的分界点：显然，$` U `$ 的最小值出现在这两种情况的分界点处。此时：

$$
C_1 = T_2 - T_1 \left\lfloor \frac{T_2}{T_1} \right\rfloor
$$

此时的利用率因子为：

$$
U = 1 - \frac{T_1}{T_2} \left[ \frac{T_2}{T_1} - \left( \frac{T_2}{T_1} - 1 \right) \left( \frac{T_1}{T_2} \right) - \left\lfloor \frac{T_2}{T_1} \right\rfloor \right]
$$

为便于表示，引入符号：  
- $` I = \left\lfloor \frac{T_2}{T_1} \right\rfloor `$  
- $` f = \left\{ \frac{T_2}{T_1} \right\} `$ （即 $` \frac{T_2}{T_1} `$ 的小数部分）

代入后，公式可表示为：

$$
U = 1 - \frac{f (1 - f)}{I + f}
$$

确定最小值：由于 $` U `$ 随 $` I `$ 单调递增，因此 $` U `$ 的最小值出现在 $` I = 1 `$ 时。对 $` f `$ 进行优化，当 $` f = 2^{1/2} - 1 `$ 时，$` U `$ 取最小值，此时：

$$
U = 2 \left( 2^{1/2} - 1 \right) \approx 0.83
$$

定理3得证。

补充说明：当 $` f = 0 `$ 时，即低优先级任务的请求周期是另一个任务请求周期的整数倍时，利用率因子 $` U = 1 `$。下面我们将推导任意数量任务的对应上界，并且讨论限制任务请求周期之间比值小于 2 的情况。

**定理 4：** 对于一组具有固定优先级顺序的 $` m `$ 个任务，并且限制任何两个请求周期的比值小于 2，处理器利用率因子的最小上界为：

$$
U = m \left( 2^{\frac{1}{m}} - 1 \right)
$$

**证明：** 设 $` \tau_1, \tau_2, \dots, \tau_m `$ 表示 $` m `$ 个任务， $` C_1, C_2, \dots, C_m `$ 为这些任务的运行时间，它们完全利用了处理器并使处理器利用率因子达到最小。假设 $` T_m > T_{m-1} > \dots > T_2 > T_1 `$。设 $` U `$ 表示处理器利用率因子。

假设：

$$
C_1 = T_2 - T_1, \quad C_2 = T_2 - T_1 + \Delta, \quad \Delta > 0
$$

令：

$$
C_1' = T_2 - T_1, \quad C_2' = C_2 + \Delta, \quad C_3' = C_3, \quad \dots, \quad C_m' = C_m
$$

显然，运行时间 $` C_1', C_2', \dots, C_{m-1}', C_m' `$ 也完全利用了处理器。设对应的利用率因子为 $` U' `$。则有：

$$
U - U' = \frac{\Delta}{T_1} - \frac{\Delta}{T_2} > 0
$$

或者，假设：

$$
C_1 = T_2 - T_1 - \Delta, \quad \Delta > 0
$$

令：

$$
C_1'' = T_2 - T_1, \quad C_2'' = C_2 - 2\Delta, \quad C_3'' = C_3, \quad \dots, \quad C_m'' = C_m
$$

同样，运行时间 $` C_1'', C_2'', \dots, C_{m-1}'', C_m'' `$ 也完全利用了处理器。设对应的利用率因子为 $` U'' `$。则有：

$$
U - U'' = -\frac{\Delta}{T_1} + \frac{2\Delta}{T_2} > 0
$$

因此，如果 $` U `$ 是最小的利用率因子，则必须有：

$$
C_1 = T_2 - T_1
$$

以类似的方法，可以证明：

$$
C_2 = T_3 - T_2, \quad C_3 = T_4 - T_3, \quad \dots, \quad C_{m-1} = T_m - T_{m-1}
$$

因此，  

$$
C_m = T_m - 2(C_1 + C_2 + \dots + C_{m-1})
$$

为了简化表示，设 

$$
g_i = \frac{T_{i+1} - T_i}{T_i}, \quad i = 1, 2, \dots, m
$$

则有：  

$$
C_i = T_{i+1} - T_i = g_i T_i - g_{i+1} T_{i+1}, \quad i = 1, 2, \dots, m-1
$$  

并且：  

$$
C_m = T_m - 2g_1 T_1
$$

最终，处理器利用率因子 $` U `$ 为：  

$$
U = \frac{C_1}{T_1} + \frac{C_2}{T_2} + \dots + \frac{C_m}{T_m}
$$

代入 $` C_i `$ 的表达式，得到：  

$$
U = \sum_{i=1}^{m-1} \left[g_i - g_{i+1} \left(\frac{T_{i+1}}{T_i}\right)\right] + 1 - 2\frac{g_1}{g_1 + 1}
$$  

化简后：  

$$
U = 1 + g_1 \frac{g_1 - 1}{g_1 + 1} + \sum_{i=2}^{m} g_i \frac{g_i - g_{i-1}}{g_i + 1}
$$

与两个任务的情况类似，当所有 $` g_i = 0 `$ 时（即所有任务的请求周期的比值为整数），处理器利用率 $` U = 1 `$。

为了找到利用率因子的最小上界，需要对公式 (4) 中的 $` g_i `$ 求导，并设导数为 0，从而解出差分方程：  

$$
\frac{\partial U}{\partial g_i} = \frac{g_i^2 + 2g_i - g_{i-1}}{(g_i + 1)^2} - \frac{g_{i+1}}{g_{i+1} + 1} = 0, \quad i = 1, 2, \dots, m-1
$$

为了简化计算，定义 $` g_0 = 1 `$。解出以上方程的通解为： 

$$
g_i = 2^{(m-i)/m} - 1, \quad i = 0, 1, \dots, m-1
$$

**结果：** 代入上述解，得到处理器利用率因子的最小上界为：  

$$
U = m\left(2^{1/m} - 1\right)
$$

- 当 $` m = 2 `$ 时，公式与直接针对两个任务的推导结果一致：  

$$
U = 2\left(2^{1/2} - 1\right) \approx 0.828
$$

- 当 $` m = 3 `$ 时：  

$$
U = 3\left(2^{1/3} - 1\right) \approx 0.78
$$

- 当 $` m \to \infty `$ 时，利用率因子趋近于：  
  
$$
U \to \ln 2 \approx 0.693
$$

扩展：在定理 4 中的限制条件，即请求周期之间的最大比值小于 2，可以被舍弃。因此，得到更一般的结果：

**定理 5：** 对于一组具有固定优先级顺序的 $` m `$ 个任务，处理器利用率因子的最小上界为：  

$$
U = m\left(2^{1/m} - 1\right)
$$

**证明：** 设 $` \tau_1, \tau_2, \dots, \tau_i, \dots, \tau_m `$ 为完全利用处理器的 $` m `$ 个任务。设 $` U `$ 表示对应的处理器利用率因子。假设对于某个 $` i `$，有 $` \left\lfloor T_m / T_i \right\rfloor > 1 `$。具体而言，设 $` T_m = qT_i + r `$，其中 $` q > 1 `$，且 $` r > 0 `$。我们用一个任务 $` \tau_i' `$ 替换任务 $` \tau_i `$，使得 $` T_i' = qT_i `$ 且 $` C_i' = C_i `$，并增加 $` C_m `$ 的运行时间，以再次完全利用处理器。此增加量至多为 $` C_i (q - 1) `$，即在任务 $` \tau_m `$ 的关键时间区间内由任务 $`\tau_i `$ 占用但未被 $` \tau_i' `$ 占用的时间。

设 $` U' `$ 表示这样一组任务的利用率因子，则有：  

$$
U' \leq U + \left[(q - 1) \frac{C_i}{T_m} \right] + \left(\frac{C_i}{T_i'} - \frac{C_i}{T_i} \right)
$$  

$$
U' \leq U + C_i (q - 1) \left[\frac{1}{qT_i + r} - \frac{1}{qT_i} \right]
$$

由于 $` q - 1 > 0 `$ 且 $`\frac{1}{qT_i + r} - \frac{1}{qT_i} < 0`$，因此 $` U' \leq U `$。  

因此，我们得出**结论：在确定处理器利用率因子的最小上界时，我们只需考虑任务集中任意两个请求周期的比值小于 2 的情况**。由 定理 4，该定理得证。

---

## 6. 放宽利用率上界  
前一节表明，在实时服务保障的要求下，对于大量任务集，其处理器利用率因子的最小上界趋近于 $` \ln(2) `$。因为需要计算任务切换的实际成本，因此找到改善这一情况的方法是有价值的。

一种简单的方法是使 $` \{T_{i+1}/T_i\} = 0 `$，其中 $` i = 1, 2, \dots, m-1 `$。然而，这并不总是可行的，另一种解决方案是对任务 $` r_m `$ 以及一些低优先级任务进行缓冲，并放宽它们的硬截止期。假设整个任务集有一个有限的周期，并且缓冲任务以某种合理的方式执行（如：先请求则先执行），那么在本文的假设条件下，可以计算出最大延迟时间和所需的缓冲量。

一种更好的解决方案是采用动态方式为任务分配优先级。本文的剩余部分将介绍一种特定的动态优先级分配方法。这种方法是最优的，因为如果某个任务集可以通过某种优先级分配调度，则它也可以通过这种方法调度。即在该方法下,处理器利用率因子 $` U `$ 的最小上界始终为 100%。

---

## 7. 截止期驱动调度算法  
我们现在研究一种动态调度算法，称为**截止期驱动调度算法**。在这种算法中，任务的优先级根据其当前请求的截止期分配。如果某任务的当前请求截止期最接近，则分配最高优先级；如果截止期最远，则分配最低优先级。在任何时刻，具有最高优先级且尚未完成的请求将被执行。这种优先级分配方法是动态的，与优先级不随时间变化的静态分配方法相对。

**定理 6：** 当使用截止期驱动调度算法在处理器上调度一组任务时，在发生溢出之前不会出现处理器空闲时间。

**证明：** 假设在溢出之前存在处理器空闲时间。具体来说，从时间 0 开始，设 $` t_3 `$ 为发生溢出的时刻，$` t_1 `$ 和 $` t_2 `$ 分别表示距离 $` t_3 `$ 最近的处理器空闲时间段的起点和终点（即在 $` t_2 `$ 和 $` t_3 `$ 之间没有处理器空闲时间）。这种情况如图 3 所示，在处理器空闲时间段 $`[t_1, t_2]`$ 之后的每个任务的第一次请求时刻分别标记为 $` a, b, \dots, m `$。

<div align="center">
  <img src="https://github.com/user-attachments/assets/43a3bc69-78cf-4a26-8f2a-c4ac33e676a7" width="600" />
</div>

假设从 $` t_2 `$ 开始，我们将任务 1 的所有请求提前，使得 $` a `$ 与 $` t_1 `$ 重合。由于在 $` t_2 `$ 和 $` t_3 `$ 之间没有处理器空闲时间，因此在 $` a `$ 被提前后，之后也不会出现处理器空闲时间。此外，溢出将发生在 $` t_3 `$ 或更早时刻。对所有其他任务重复相同的论证，可以得出结论：如果所有任务都从 $` t_1 `$ 开始，则会发生溢出且不会出现处理器空闲时间。然而，这与假设从时间 0 开始到溢出之前存在处理器空闲时间相矛盾。定理6得证。

下面用**定理 6**来证明**定理 7**：

**定理 7：** 对于给定的 $` m `$ 个任务集，截止期驱动调度算法是可行的，当且仅当以下条件成立：  

$$
\frac{C_1}{T_1} + \frac{C_2}{T_2} + \dots + \frac{C_m}{T_m} \leq 1
$$

**证明：**

**必要性：** 所有任务在 $` t = 0 `$ 到 $` t = T_1T_2 \dots T_m `$ 之间的计算时间总需求可表示为：  

$$
(T_2T_3 \dots T_m)C_1 + (T_1T_3 \dots T_m)C_2 + \dots + (T_1T_2 \dots T_{m-1})C_m
$$

如果总需求超过可用处理器时间，即：  

$$
(T_2T_3 \dots T_m)C_1 + (T_1T_3 \dots T_m)C_2 + \dots + (T_1T_2 \dots T_{m-1})C_m > T_1T_2 \dots T_m
$$

或等价于：  

$$
\frac{C_1}{T_1} + \frac{C_2}{T_2} + \dots + \frac{C_m}{T_m} > 1
$$

显然，此时不存在可行的调度算法。

**充分性：** 假设以下条件成立：  

$$
\frac{C_1}{T_1} + \frac{C_2}{T_2} + \dots + \frac{C_m}{T_m} \leq 1
$$

但调度算法仍不可行。这意味着在 $` t = 0 `$ 和 $` t = T_1T_2 \dots T_m `$ 之间存在溢出。此外，根据 **定理 6**，在 $` t = 0 `$ 和 $` t = T `$ 之间（其中 $` 0 < T < T_1T_2 \dots T_m `$）存在一个 $` t = T `$，在此时刻发生溢出，并且 $` t = 0 `$ 和 $` t = T `$ 之间没有处理器空闲时间。

如图 4 所示情况，设 $` a_1, a_2, \dots `$ 表示在 $` T `$ 之前任务的请求时间，且这些请求的截止期在 $` T `$ 时完成； $` b_1, b_2, \dots `$ 表示在 $` T `$ 之前任务的请求时间，且这些请求的截止期晚于 $` T `$。则有以下两种情况需要分析。

<div align="center">
  <img src="https://github.com/user-attachments/assets/b0088194-ca44-4374-9271-5d96bf2d9a20" width="600" />
</div>

**情况 1：** 在 $` b_1, b_2, \dots `$ 处请求的任务在 $` T `$ 前未被执行。

在这种情况下，从 0 到 $` T `$ 的计算时间总需求为：  

$$
\left\lfloor \frac{T}{T_1} \right\rfloor C_1 + \left\lfloor \frac{T}{T_2} \right\rfloor C_2 + \dots + \left\lfloor \frac{T}{T_m} \right\rfloor C_m
$$

由于没有处理器空闲时间：  

$$
\left\lfloor \frac{T}{T_1} \right\rfloor C_1 + \left\lfloor \frac{T}{T_2} \right\rfloor C_2 + \dots + \left\lfloor \frac{T}{T_m} \right\rfloor C_m > T
$$

同时，因为对于所有 $` x `$，有 $` x \geq \left\lfloor x \right\rfloor `$，因此：  

$$
\frac{T}{T_1} C_1 + \frac{T}{T_2} C_2 + \dots + \frac{T}{T_m} C_m > T
$$

等价于：  

$$
\frac{C_1}{T_1} + \frac{C_2}{T_2} + \dots + \frac{C_m}{T_m} > 1
$$

这与假设条件 $`(C_1/T_1) + (C_2/T_2) + \dots + (C_m/T_m) \leq 1`$ 矛盾。

**情况 2：** 在 $` b_1, b_2, \dots `$ 处请求的某些任务在 $` T `$ 前已被部分执行。

由于在 $` T `$ 时发生溢出，因此必定存在一个时刻 $` T' `$，满足在区间 $` T' < t < T `$ 内，$` b_1, b_2, \dots `$ 处的请求未被执行。换句话说，在 $` T' < t < T `$ 内，仅执行那些截止期为 $` T `$ 或更早的请求，如图 5 所示。

<div align="center">
  <img src="https://github.com/user-attachments/assets/18f09477-654b-4aba-b12b-57300d95c47f" width="600" />
</div>


此外，由于 $` b_1, b_2, \dots `$ 处请求的任务直到 $` T' `$ 时仍在执行，这意味着所有在 $` T' `$ 前启动且截止期在 $` T `$ 或更早的请求都在 $` T' `$ 前完成。因此，在 $` T' < t < T `$ 内，处理器时间的总需求为：  

$$
\left\lfloor \frac{T - T'}{T_1} \right\rfloor C_1 + \left\lfloor \frac{T - T'}{T_2} \right\rfloor C_2 + \dots + \left\lfloor \frac{T - T'}{T_m} \right\rfloor C_m
$$

由于 $` T `$ 时发生溢出：  

$$
\left\lfloor \frac{T - T'}{T_1} \right\rfloor C_1 + \left\lfloor \frac{T - T'}{T_2} \right\rfloor C_2 + \dots + \left\lfloor \frac{T - T'}{T_m} \right\rfloor C_m > T - T'
$$

已知 $` x \geq \left\lfloor x \right\rfloor `$，有：  

$$
\frac{C_1}{T_1} + \frac{C_2}{T_2} + \dots + \frac{C_m}{T_m} > 1
$$

这再次与假设条件 $`(C_1/T_1) + (C_2/T_2) + \dots + (C_m/T_m) \leq 1`$ 矛盾。


因此，当假设条件成立时，截止期驱动调度算法是可行的。反之亦然，定理 7得证。

**综上所述，截止期驱动调度算法是最优的动态调度算法。若某任务集可以通过某种调度算法进行调度，则也可以通过截止期驱动调度算法调度。**

## 8. 混合调度算法  
在本节中，将研究一类混合了速率单调调度和截止期驱动调度的调度算法。我们称该类算法为**混合调度算法**。

研究混合调度算法的动机源于以下观察：当今计算机的中断硬件充当固定优先级调度器，与动态硬件调度器不兼容。另一方面，对于低速任务来说，如果采用截止期驱动调度代替固定优先级分配，其软件调度实现的成本并不会显著增加。

具体而言，设任务 $`1, 2, \dots, k`$ 是周期最短的 $`k`$ 个任务，它们按照固定优先级的速率单调调度算法进行调度；剩余任务 $`k+1, k+2, \dots, m`$ 在处理器未被任务 $`1, 2, \dots, k`$ 占用时，按照截止期驱动调度算法进行调度。

**可用性函数定义：** 设 $`a(t)`$ 是 $`t`$ 的非递减函数。如果对于所有 $`T`$，满足 $`a(T) < a(t + T) - a(t)`$，则称 $`a(t)`$ 是次线性的。  

一组任务的处理器可用性函数定义为从 0 到 $`t`$ 的累积处理器时间，该时间对这组任务是可用的。假设 $`k`$ 个任务已由固定优先级调度算法调度。令 $`a_k(t)`$ 表示处理器对任务 $`k+1, k+2, \dots, m`$ 的可用性函数。显然，$`a_k(t)`$ 是 $`t`$ 的非递减函数。此外，通过关键时间区间的论证，可以证明 $`a_k(t)`$ 是次线性的。

**定理 8：** 如果一组任务在一个可用性函数 $`a_k(t)`$ 次线性的处理器上使用截止期驱动调度算法进行调度，则在发生溢出之前不会出现处理器空闲时间。

**证明：** 其证明可参考定理 6 的证明。

**定理 9：** 对于具有可用性函数 $`a_k(t)`$ 的处理器，截止期驱动调度算法的可行性必要且充分的条件是：  

$$
\left\lfloor \frac{t}{T_{k+1}} \right\rfloor C_{k+1} + \left\lfloor \frac{t}{T_{k+2}} \right\rfloor C_{k+2} + \dots + \left\lfloor \frac{t}{T_m} \right\rfloor C_m \leq a_k(t)
$$

对所有 $`t`$ 都成立，且 $`t`$ 是 $`T_{k+1}, T_{k+2}, \dots, T_m`$ 的倍数。

**证明：**

**必要性：** 在任何时刻，处理器时间的总需求不能超过处理器的总可用时间。因此必须满足：  
对于任意 $`t`$ ，不等式

$$
\left\lfloor \frac{t}{T_{k+1}} \right\rfloor C_{k+1} + \left\lfloor \frac{t}{T_{k+2}} \right\rfloor C_{k+2} + \dots + \left\lfloor \frac{t}{T_m} \right\rfloor C_m \leq a_k(t)
$$

成立。

**充分性：** 假设满足定理中的条件，但在 $`T`$ 时发生溢出。我们分析定理 7 中的两种情况：

**情况 1：** 在 $`T`$ 时发生溢出，满足：  
$$
\left\lfloor \frac{T}{T_{k+1}} \right\rfloor C_{k+1} + \left\lfloor \frac{T}{T_{k+2}} \right\rfloor C_{k+2} + \dots + \left\lfloor \frac{T}{T_m} \right\rfloor C_m > a_k(T)
$$  
这与假设矛盾。注意 $`T`$ 是 $`T_{k+1}, T_{k+2}, \dots, T_m`$ 的倍数。

**情况 2：** 存在一个时刻 $`T'`$，使得在区间 $`T' < t < T`$ 内，$`b_1, b_2, \dots`$ 的请求未被执行。在此区间内，仅执行截止期为 $`T`$ 或更早的请求。进一步假设 $`T - T' = e`$ 是 $`T_{k+1}, T_{k+2}, \dots, T_m`$ 的倍数。

由于： 

$$
\left\lfloor \frac{T - T'}{T_{k+1}} \right\rfloor = \left\lfloor \frac{T - T' - e}{T_{k+1}} \right\rfloor
$$

可得：  

$$
\left\lfloor \frac{T - T'}{T_{k+1}} \right\rfloor C_{k+1} + \dots + \left\lfloor \frac{T - T'}{T_m} \right\rfloor C_m > a_k(T - T') > a_k(T - T' - e)
$$

这与假设矛盾。因此，充分性得证。

虽然**定理 9** 的结果是一个有用的通用结论，但其应用需要解决大量的不等式。在特定情况下，推导调度可行性的充分条件可能比直接使用 **定理 9** 更加有利。举例来说，考虑一种特殊情况：三个任务由混合调度算法调度，其中周期最短的任务被分配固定且最高优先级，其余两个任务则通过截止期驱动调度算法调度。

可以很容易验证，如果满足以下条件，则混合调度算法是可行的：  

$$
1 - \frac{C_1}{T_1} - \min\left[\frac{T_2 - C_1}{T_2}, \frac{C_3}{T_3}\right] > \frac{C_2}{T_2} + \frac{C_3}{T_3}
$$

同样可以验证，如果满足以下条件，则混合调度算法也是可行的：  

$$
C_2 \leq a_1(T_2), \quad \left\lfloor \frac{T_3}{T_2} \right\rfloor C_2 + C_3 \leq a_1\left(\left\lfloor \frac{T_3}{T_2} \right\rfloor T_2\right), 
$$  

$$
\left(\left\lfloor \frac{T_3}{T_2} \right\rfloor + 1\right)C_2 + C_3 \leq a_1(T_3)
$$

这些陈述的证明主要涉及一些相对直接但冗长的不等式操作，可参考 Liu [13]。遗憾的是，这些充分条件对应的处理器利用率远低于 **定理 9** 中的必要且充分条件。

---

## 9. 比较与评论  
**定理 9** 中的约束表明，混合调度算法并不能普遍实现 100% 的处理器利用率。以下简单的例子可以说明这一点：

设 $`T_1 = 3`$、$`T_2 = 4`$、$`T_3 = 5`$，且 $`C_1 = C_2 = 1`$。由于 $`a_1(20) = 13`$，可以很容易看出 $`C_3`$ 的最大允许值为 2。此时对应的利用率因子为： 

$$
U = \frac{1}{3} + \frac{1}{4} + \frac{2}{5} = 98.3\%
$$

如果这三个任务由截止期驱动调度算法调度，则 $`C_2`$ 可增加至 2.0833...，实现 100% 的利用率。  
如果这三个任务全部由固定优先级速率单调调度算法调度，则 $`C_3`$ 被限制为 1 或更小，对应的利用率因子最大为：  

$$
U = \frac{1}{3} + \frac{1}{4} + \frac{1}{5} = 78.3\%
$$

这一值仅略高于三个任务的固定优先级利用率下界。

虽然尚未找到混合调度算法的处理器利用率最小上界的封闭形式表达式，但这一例子表明，混合调度算法的利用率上界比固定优先级速率单调调度算法的上界要宽松得多。因此，混合调度算法可能适用于许多应用场景。

---

## 10. 结论  
本文的前部分提出了定义分析环境的五个假设，其中最重要但最难以完全成立的是假设 (A1) 和 (A4)：即所有任务具有周期性请求，且运行时间是恒定的。如果这些假设不成立，则每个任务的关键时间区间应被定义为从请求到截止期之间的时间区间，在此期间具有较高优先级的任务可能执行的最大计算量。

如果任务的运行时间和请求周期的详细信息未知，则必须基于假定的周期性和恒定运行时间来计算任务运行时间的可行性约束。周期应取为最短请求间隔，运行时间应取为最长运行时间。在这种情况下，本文的分析结果将不再有效，并且任务的非周期性可能会对处理器利用率施加严格限制。

当前的固定优先级顺序应与每个任务的请求和截止期之间的最短时间间隔单调相关，而不是与未定义的请求周期相关。如果某些任务的截止期比假设 (A2) 中更紧，尽管仅涉及最高优先级任务时对利用率的影响较小，这一原则仍适用。

可以认为 (A1) 和 (A4) 的含义足够重要，应作为任何需要保障服务的实时任务的设计目标。

---

## 总结：  
本文探讨了与硬实时环境中的多编程相关的一些问题，此类环境通常以过程控制和监测为代表。在本文假设的环境下：

1. 一种将任务优先级按请求速率单调分配的调度算法被证明是固定优先级调度算法中最优的。对于大任务集，该算法的处理器利用率最小上界约为 70%。  
2. 动态截止期驱动调度算法被证明是全局最优的，能够实现 100% 的处理器利用率。  
3. 混合调度算法结合了两种调度算法的特点，既实现了截止期驱动调度算法的大部分优势，又可以在现有计算机上容易实现。

---
