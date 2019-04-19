## 前言
**by Turner Whitted and Martin Stich**

简单，并行，易用。这些都是光线追踪的主题。我从不认为光线追踪是实现全局光照的最好工具，但由于它的简单，使得其具有持续的吸引力。 很少有图形渲染算法这么易于可视化，讲解或编码。这种简单性使刚入门的程序员可以轻松渲染几个透明球体和由点光源照明的棋盘。 在现代实践过程中，路径追踪和其他的脱离原始算法的实现都有一点复杂，但它们仍然只限于简单的直线与沿途的任何东西相交。

“令人难堪的并行”一词早在没有任何合理的并行引擎来运行光线追踪之前就已经应用于光线追踪了。如今，光线追踪拥有了与之相匹配的拥有惊人的并行和原始计算能力的现代GPU。

对于所有程序员来说，易用一直是都是一个问题。几十年前，如果一台电脑做不到我想要它做的事情，我会在它后面走来走去，对电路做一些小小的改动。(我不是开玩笑的)。再往后的几年，甚至在图形API层的底层添加定制都是不可想象的。几十多年前，随着可编程着色的逐步发展，这种情况发生了微妙的变化。如今，GPU的灵活性连同支持的编程工具,使我们能够充分的利用并行处理元件的全部计算潜力。

那么，如何实现实时光线追踪的呢？显然，在性能、复杂性和准确性方面的挑战无法阻止图形程序员同时提高质量和速度。图形处理器也在不断发展，因此光线跟踪不再是圆孔中的方形钉。在图形硬件中引入显式的光线追踪加速的特性，是实现实时光线追踪应用的重要一步。将光线追踪的简单性，固有的并行性，和现代GPU提供的易用性和功率结合，使得的实时光线追踪性能，可以满足每个图形程序员的需求。 然而，正如获得驾驶执照并不像赢得汽车比赛那样。 有一些要学习的技巧。有一些共享的经验。与任何学科一样，当为本文做出贡献的专家分享这些技巧时，它们就会成为真正的精粹。

—Turner Whitted   December 2018  

对于图形学来说，这是一个令人激动的时刻！我们已经进入到实时光线追踪的时代——每个人都知道这个时代会到来，但是直到最近，我们还认为这将会在几年或者十几年之后才能到来。上一次我们的行业经历这样的“大爆炸”事件是在2001年，当时第一个支持可编程着色的硬件和API为开发者提供了无限的可能。可编程着色技术促进了非常多的渲染技术的发明，很多书籍都介绍了这些例如我们这本书（其他例如：Real-Time Rendering 和 GPU Gems等等）。这些技术背后越来越多的独创性，加上GPU不断增长的功率和通用性，是过去几年实时图形技术进步的主要驱动力。得益于这些进步，游戏和其他的图形应用变得更加的漂亮。

可是，尽管现在仍在取得进步，但是，基于光栅化的渲染已经到达了瓶颈。尤其是，当我们涉及到模拟灯光行为的时候（真实感渲染的本质），这些改进已经达到了收益递减的程度。这是因为，任何形式的光传输模拟所需要的基本操作，光栅化都无法提供:这项能力是能够从场景中的任何给定的点来询问“我周围有什么?”。由于这非常的重要，过去几十年的发明的大多数重要的光栅化技术的核心都是针对这种限制的巧妙的变通方法。他们通常采用的方法是预先生成包含场景大致信息的一些数据结构，然后在着色期间执行对该结构的查找。

阴影贴图，光照烘焙贴图，屏幕空间反射，AO，光照探针，体素网格都是这种变通方案的例子。他们共同面临的问题是，他们所依赖的辅助数据结构的精确度是有限的。这些数据结构必然只包含简化的表示，因为除了最简单的情况外，所有的情况下都不可能预先计算并按精确结果所需的数量和分辨率存储它们。因此，基于这些数据结构的技术都有不可避免的失败案例，这些失败案例会导致明显的rendering artifacts或完全失去效果。这就是为什么相关阴影（Contact Shadows）看起来并不是完全正确，相机后面的对象在反射中丢失，间接光的细节很粗糙等等。此外，这些技术都需要手动调参，才能得到最好的效果。

进入到光线追踪，光线追踪能够优雅并准确的解决这些问题，因为它精确的提供了光栅化技术试图去模拟的基本操作：允许我们从场景中的任何地方向我们喜欢的任何方向发出一个查询，并找出哪个对象、在哪个位置、距离多远被击中。这可以通过检查场景中实际的几何体实现这一点，而不限于近似值。因此，基于光线跟踪的计算足够精确，可以在非常精细的层次上模拟各种光传输。当我们的目标是照片级别的时候，我们需要确定光子在虚拟世界中传播的复杂路径，这种能力是其他方法无可替代的。光线追踪是真实感渲染的基本组成部分，这就是为什么它引入实时领域对于计算机图形学来说是如此重要的一步。

当然，使用光线追踪生成图形并不是一个新鲜的想法。它的起源可以追溯到20世纪60年代，电影渲染和设计可视化等应用程序几十年来一直依赖它来产生逼真的效果。那么，最新的发展是在现代系统中光线的处理速度。由于采用了专用的用的光线追踪芯片，通过测算，最近引入的NVIDIA Turing GPU的吞吐量达到每秒数十亿光线的速度，比上一代提高了一个数量级。实现这种性能水平的硬件称为RT Core，这是一个经过多年研究和开发的复杂单元。 RT Core与GPU上的流式多处理器（SMs）紧密耦合，并实现光线追踪操作的关键“内循环”： 层次包围盒（BVH）的遍历和光线与三角形的相交检验。在专用电路中执行这些计算不仅比软件实现更快，而且在并行处理光线的同时释放通用SM内核来执行其他工作，例如着色。同时并行处理光线。 通过RT Cores实现的性能的巨大飞跃为光线追踪在要求实时应用中成为可行的奠定了基础。

想要应用程序，尤其是游戏，能够有效的利用RT Cores,还需要创建新的API，这些API可以无缝的继承到已经建立的ecosys tems中。在和微软的密切合作下，DirectX光线跟踪（DXR）开发成为DirectX 12不可或缺的一部分。这将在第3章提供了一个介绍。 Vulkan的NV_ray_tracing扩展在Khronos API暴露了相同的概念。

快速光线追踪的GPU和API现在已经被广泛使用，并为图形程序员的工具箱添加了强大的新工具。然而，这绝不意味着实时的图形是一个已解决的问题。 实时的应用程序无法满足的帧率要求转化为光线预算，这些预算很少以至于无法通过强力手段轻松解决所有的光传输模拟问题。与光栅化技术的技巧多年来的进步不同，我们将看到一种巧妙的光线跟踪技术的不断发展，这将缩小实时性能与离线渲染的“像素”质量之间的差距。 其中一些技术将建立在非实时生产渲染领域的丰富经验和研究基础之上。 其他的将实时应用程序需求的特殊需求，如游戏引擎等。 Epic，SEED和NVIDIA的图形工程师在第一批基于DXR的演示中推演出两个很好的案例研究，这些可以在第19章和第25章中找到。

作为有幸在创建NVIDIA光线跟踪技术方面发挥作用的人，最终能够在2018年推出它对我来说是一次非常有意义的经历。在几个月内，实时光线跟踪从一个研究领域转变为消费产品，主流GPU的专用硬件中完成了独立于供应商的API支持，以及“EA”的战地， 首个以光线跟踪效果为标题的3A游戏。游戏引擎供应商采用光线追踪的速度，以及我们从开发人员那里看到的热情程度，都超出了我们的预期。很明显，人们强烈希望将实时图像的质量提高到只有光线跟踪才能达到的水平，这反过来激励我们NVIDIA继续推进这项技术。事实上，图形仍然处于光线跟踪时代的开始：未来十年将看到更强大的GPU，算法的进步，人工智能在渲染的更多方面的结合，以及用于光线跟踪的全新游戏引擎。在图形足够之前还有很多工作要做，而且有助于实现下一个里程碑的工具之一，就是本书。

Eric Haines和Tomas Akenine-M'oller是图形学的资深人士，他们的工作已经为开发人员和研究人员提供了数十年的教育和启发。 在这本书中，他们将重点放在光线追踪，因为这项技术获得了前所未有的势头。来自全行业的一些顶级专家在本书中分享了他们的知识和经验，为社区创造了宝贵的资源，将对图形的未来产生持久的影响。

                                                ——Martin Stich
                    DXR & RTX Raytracing Software Lead, NVIDIA
                                                December 2018

