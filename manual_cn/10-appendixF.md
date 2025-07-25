---
sidebarDepth: 3
---

# 附录F. 配置文件

该章节不会对所有配置选项都做解释，而是针对其中可能有用的一些选项进行分析与解释。这里按照配置文件中的配置参数呈现的顺序进行描述（而非它们在 RTKNAVI/RTKPOST 菜单中显示的顺序），不过这些参数对于两者都是适用的。与第3章中提到的配置描述相比，这里的会更详细，并且包含一些调试经验。

后文中的参数值是用于 5 Hz 的流动站（rover）研究的配置参数。同样的配置文件可用于 RTKNAVI、RTKRCV、RTKPOST 或 RNX2RTKP。

下面以<font style="color: #1560BD;">蓝色</font>高亮显示的设置和选项仅适用于 demo5 代码，而不适用于 RTKLIB 原始版本。但除此之外，下面描述的大部分内容适用于这两种代码。demo5 作者是基于 Ublox M8T 和 F9P 接收机进行 GNSS-RTK 研究的，并且主要应用于短基线环境，因此这些设置更适用于这些低成本接收机。

## SETTING1
### pos1-posmode = static, kinematic, <font style="color: #1560BD;">static-start</font>, movingbase, fixed
- static：如果 rover 是静止的，使用“static”；
- kinematic：如果它是移动的，使用“kinematic”（或“static-start”）；
- static-start：“static-start”假设 rover 在第一次 Fix 完成之前是静止的，然后切换到动态模式，程序允许滤波器利用 rover 最开始是静止的先验信息；
- movingbase：如果 base 和 rover 都在移动，那么可以使用“movingbase”模式（通常用来定姿）。“movingbase”模式与 dynamics 模式不兼容，请不要同时启用两者。如果 base 与 rover 保持固定距离，在 movingbase 模式下设置“pos2-baselen”和“pos2-basesig”；
- fixed：如果你知道 rover 的确切位置，并且只对分析残差感兴趣，可以使用“fixed”。  

### pos1-frequency = L1, L1+L2, L1+L2+L5
- L1：单频接收机使用“L1”；
- L1+L2/E5b：若 rover 使用 L2 GPS/GLONASS/Beidou and/or Galileo E5b，则用“L1+L2/E5b”；
- L1+L2+L5：“L1+L2+L5”则表示 rover 还会使用 L5 GPS/GLONASS/Bediou and/or Galileo E5a 数据

注意这其中不包含 L1+L5 的单独形式，另外 RTKLIB 对 Beidou 的 B2a 频点还没有很好的支持（demo5 b34h版本开始支持B2a了）。

### pos1-soltype = forward, backward, combined
滤波类型表示卡尔曼滤波在后处理中运行的时间方向。“combined”模式首先向前运行过滤器，然后向后运行并组合结果。对于每个历元：
- 如果两个方向都有Fix，那么组合的结果是两个Fix状态的平均值，除非两者之间的差异太大，在这种情况下状态将是float的。
- 如果只有一个方向有Fix，则使用该值固定状态。
- 如果两个方向都是float的，那么平均值将被使用，状态将是float的。

使用“combined”后的结果并不总是更好，因为在任何一个方向上运行时错误的Fix通常会导致合并后的结果是float或异常解。“combined”的主要优点是，它通常会为您提供数据开头的固定状态，而仅向前的解决方案需要一些时间来收敛。

在“combined”模式下，偏差状态在开始向后运行之前被重置，以最大限度地提高向前和向后解的独立性。在“combined mode-no phase reset”中，不重置偏置状态可以避免在反向解开始时需要重新收敛。只有在调试时遇到初始定位困难，想要了解正确的卫星相位偏差时，才使用“backward”设置。

### pos1-elmask = 15 (degrees)
用于计算位置的最小卫星高度，通常将其设置为 10-15 度，以减少多径效应进入解算的概率，但这个设置取决于 rover 的环境。天空视野越开阔，这个值可以设置得越低。低仰角卫星还会带来较大的大气误差，这也是排除最低仰角卫星的另一个原因。

对于城市峡谷环境，最好将仰角设高。另外在往下的角度范围内，固定率与仰角的设置并不具备可参考的规律。

### pos1-snrmask-r, pos1-snrmask-b = off,on
用于计算位置的 rover（-r）和 base（-b）的最小卫星信噪比（SNR）。最优值会因接收机类型和天线类型而异，不过在处理较具挑战性的数据集时，最好启用它以移除低质量卫星，从而改善结果。

### pos1-snrmask_L1 =35,35,35,35,35,35,35,35,35
为每 5 度仰角设置信噪比（SNR）阈值。可以将所有值设为相同，根据标称信噪比选择30到38分贝之间的某个值。这些值仅在“pos1-snrmask_x”设置为“on”时使用。如果使用双频，你还需要设置“pos1-snrmask_L2”和/或“pos1-snrmask_L5”。

### pos1-dynamics = on
将 rover 设置为 dynamic 会在滤波器中为 rover 添加速度和加速度状态。这将改善“kinematic”和“static-start”模式的结果，但对“static”模式没有影响。确保根据 rover 的加速度特性适当设置“prnaccelh”和“prnaccelv”。dynamic 与“movingbase”模式不兼容，因此在使用该模式时需将其关闭。

### pos1-posopt1 = off, on (Sat PCV)
设置是否使用卫星天线相位中心变化。对于RTK（实时动态定位），建议关闭此选项，因为天线偏移会相互抵消。对于PPP（精密单点定位），建议启用，但如果设置为“on”，请确保在文件参数中指定卫星天线PCV（相位中心变化）文件。

### pos1-posopt2 = off, on (Rec PCV)
设置是否使用接收机天线相位中心变化。如果设置为“on”，你需要在文件参数中指定接收机天线PCV（相位中心变化）文件，并在天线部分指定 base 和 rover 使用的接收机天线类型。只有测量级天线才包含在IGS提供的天线文件中，因此仅当你的天线在该文件中时才使用此选项。它主要影响z轴的精度，因此如果你关注高程，这可能很重要。如果 base 和 rover 的天线相同，可以关闭此选项，因为它们会相互抵消。

### pos1-posopt5 = off, on (RAIM FDE)
如果某颗卫星的残差超过阈值，该卫星将被排除。这只会排除误差非常大的卫星，但需要相当多的计算量，因此通常将其禁用。

### pos1-exclsats=
如果你知道某颗卫星有问题，可以通过在此列出将其从解算中排除。demo5作者仅在极少数情况下用于调试，当demo5作者怀疑某颗卫星有问题时才会使用。你还可以通过在卫星编号前加“+”来强制使用一颗不健康的卫星。

使用示例：
```text
# 使用空格分隔字符串
pos1-exclsats = G01 G02 R03
```

### pos1-navsys = 7, 15,
尽量启用所有可用的卫星系统，因为更多观测信息通常更好。例外情况是，如果使用EGNOS卫星，则不应启用SBAS（卫星增强系统）。

## SETTING2
### pos2-armode = continuous, fix-and-hold
整周模糊度解算策略。“continuous”模式不利用固定解来调整相位偏差状态，所以它对假固定最不敏感。“fix-and-hold”则会利用固定解的反馈来帮助跟踪模糊度。对于低成本接收机，可以使用“fix-and-hold”并将跟踪增益（pos2-varholdamb）调整得足够低，以最大限度地减少假固定的可能性。如果“armode”未设置为“fix-and-hold”，则下面提到的和 hold 相关的任何选项都不再适用。

### <font style="color: #1560BD;">pos2-varholdamb=0.1, 1.0 (meters)</font>
通过该参数来调整fix-and-hold模式下的跟踪增益。实际上，它是一个方差（对应EKF中R阵的对角线数值），而不是增益，因此较大的值会导致增益较低。

默认值为0.1，任何超过100的值基本都会让fix-and-hold模式失效。这个值被作用在hold过程中生成的伪测量的方差（浮点解算之后进行的一次新的量测更新），这些伪测量为卡尔曼滤波器中的偏差状态提供反馈，促使其向整数值靠拢。将这个值设置在0.1到1.0之间，既能提供足够的增益来辅助跟踪，又能避免在大多数情况下跟踪到错误的固定解。

### pos2-gloarmode = on, <font style="color: #1560BD;">fix-and-hold, autocal</font>
GLONASS卫星的整数模糊度解算。如果你的接收机类型相同，或者两者的GLONASS硬件偏差均为零，你可以将此参数设置为“on”。如果你的接收机偏差不同，则需要考虑通道间偏差。最简单的处理方法是将此参数设置为“fix-and-hold”。在这种情况下，GLONASS卫星在通道间偏差校准完成之前不会用于模糊度解算，而通道间偏差的校准从第一次hold开始。作为另一种选择，你可以将此参数设置为“autocal”，然后通过“pos2-arthres2”参数指定基准站和移动站之间的差分硬件偏移。这将允许GLONASS卫星立即用于模糊度解算，因此通常比“fix-and-hold”设置表现更好。此外，“autocal”功能还可以通过迭代方法，使用零基线或短基线来确定通道间偏差。

延伸博客：https://blog.csdn.net/weixin_42918498/article/details/119118410

### <font style="color: #1560BD;">pos2-gainholdamb=0.01</font>
在demo5代码中，可以通过这个参数来调整GLONASS卫星通道间偏差校准的增益。

### pos2-arthres = 3.0
用于判断模糊度固定解是否有足够的置信度（比率阈值），从而确定固定解。它是次优解的残差平方与最优解的残差平方的比值（ratio-test），通常将比值保持在默认值3.0，并同时调整其他参数。

尽管较大的Ratio阈值表示比较低的阈值具有更高的置信度，但两者之间并没有确定的关系。卡尔曼滤波器状态中的误差越大，对于给定的AR比值，对该解的置信度就越低。一般来说，卡尔曼滤波器的误差在首次收敛时最大，因此此时最有可能出现错误的固定解。降低pos2-arthres1参数可以帮助避免这种情况。

### <font style="color: #1560BD;">pos2-arthresmin, pos2-arthresmax=1.5, 10</font>
如果这些值被设置为等于pos2-arthres，那么模糊度解算阈值将被固定。否则，阈值将根据用于模糊度解算的卫星数量进行调整。对于8对卫星的情况，将使用标准值，随着卫星数量的增加，阈值会降低，而随着卫星数量的减少，阈值会增加。调整后的值将被最小和最大阈值限制所限制。调整率基于FFRT方法所使用的值，demo5的实现过程仅根据卫星数量进行调整，而不考虑模型强度。

### <font style="color: #1560BD;">pos2-arfilter = on</font>
将此“arfilter”参数设置为“on”将允许对新卫星或从周跳中恢复的卫星进行有效性检验。如果一颗卫星在首次加入时显著恶化了模糊度比率（AR ratio），那么它在模糊度解算中的使用将被延迟。启用此功能可以让你减少对“arlockcnt”参数的使用，该参数虽然也有类似的作用，但它是通过固定的延迟计数来实现的，不是很灵活。

### <font style="color: #1560BD;">pos2-arthres1 = 0.004-0.10</font>
整数模糊度解算将延迟到位置状态的方差达到此阈值时才开始进行。其目的是避免在滤波器中的偏差状态尚未收敛之前出现的假固定。如果你将“eratio1”设置为大于100的值，并且使用单星座或单频解，那么将此参数设置为相对较低的值尤为重要。如果你在解算结果中看到模糊度比率（AR ratio）为零的情况持续过长，那么你可能需要增加这个阈值，因为这意味着由于未达到阈值，模糊度解算被禁用了。demo5作者发现0.004到0.10的值通常对一些低成本接收机很有效，但如果你的测量数据质量较低，你可能需要增加这个值，以避免首次固定解的延迟过长，或者在多次发生周跳后丢失固定解。

### <font style="color: #1560BD;">pos2-arthres2</font>
按频率槽计算的相对GLONASS硬件偏差（m）。仅当pos2-gloarmode设置为“autocal”时，才会使用此参数，用于指定两个不同接收机制造商之间的通道间偏差。有关常见接收机类型的合适值，以及如何使用此参数进行迭代搜索以找到未指定接收机类型的值，请参阅[这篇帖子](https://rtklibexplorer.wordpress.com/2018/06/14/glonass-ambiguity-resolution-with-rtklib-revisited/)。

### <font style="color: #1560BD;">pos2-arthres3 = 1e-9,1e-7</font>
GLONASS硬件偏差状态的初始方差。仅当pos2-gloarmode设置为“autocal”时，才会使用此参数。较小的值会使pos2-arthres2中指定的初始值获得更大的权重。当pos2-arthres2设置为已知偏差时，demo5作者使用1e-9；而在进行迭代搜索时，demo5作者使用1e-7。

### <font style="color: #1560BD;">pos2-arthres4 = 0.00001,0.001</font>
GLONASS硬件偏差状态的卡尔曼滤波器过程噪声。较小的值会使pos2-arthres2中指定的初始值获得更大的权重。当pos2-arthres2设置为已知偏差时，demo5作者使用0.00001；而在进行迭代搜索时，demo5作者使用0.001。

### pos2-arlockcnt = 0, 5(*sample rate)
在将新卫星或从周跳中恢复的卫星用于整数模糊度解算之前，需要延迟的采样数。这可以避免因包含尚未收敛的卫星而导致模糊度比值（AR ratio）被破坏。需与“arfilter”配合使用。请注意，单位是采样数，而不是时间单位，因此如果你更改了移动站测量的采样率，可能需要对其进行调整。demo5作者通常将此参数设置为零，因为u-blox接收机在标记可疑观测值方面表现非常好，但对于其他接收机，demo5作者会将其设置得更高。

### <font style="color: #1560BD;">pos2-minfixsats = 4</font>
获得固定解所需的最小卫星数量。用于避免因极少数卫星而导致的假固定，尤其是在频繁发生周跳的期间。

### <font style="color: #1560BD;">pos2-minholdsats = 5</font>
整数模糊度结果进入 hold 阶段所需的最小卫星数量。用于避免因极少数卫星而导致的错误保持，尤其是在频繁发生周跳的期间。

### <font style="color: #1560BD;">pos2-mindropsats = 10</font>
启用每历元排除单颗卫星进行模糊度解算所需的最小卫星数量。在每个历元中，会排除一颗不同的卫星。如果排除某颗卫星后，模糊度比值显著改善，则该卫星将从用于模糊度解算的卫星列表中移除。

### <font style="color: #1560BD;">pos2-rcvstds = on,off</font>
实验性功能。启用此功能后，基于接收机报告的测量值的标准差，调整原始伪距和相位测量观测的方差。目前此功能仅支持u-blox接收机。方差的调整是基于卫星高度角的stats-errphaseel参数调整之外的。demo5作者通常在关闭此功能时获得更好的结果。

### pos2-arelmask = 15
功能上与默认值0没有区别，因为小于“elmask”的高度角不会用于模糊度解算，但demo5作者将其更改以避免混淆。

### pos2-arminfix = 20-100 (5-20*sample rate)
模糊度进入hold所需的连续固定样本数量。增加此值可能是减少错误hold的最有效方法，但同时也会增加首次hold的时间和重新获得hold的时间。随着模糊度跟踪增益的降低（即pos2-varholdamb增加）和观测数量的增加，可以减少arminfix。请注意，如果移动站测量的采样率发生变化，可能需要调整此值。

### pos2-elmaskhold = 15
功能上与默认值0没有区别，因为小于“elmask”的高度角不会用于保持模糊度解算结果，但demo5作者将其更改以避免混淆。

### pos2-aroutcnt = 100 (20*sample rate)
导致模糊度重置的连续缺失样本数量。该参数与流动站的采样率有关，因此如果采样率发生变化，则需要调整此值。

### pos2-maxage = 100
该指标可以理解为来自基站差分数据的时效性，定义为流动站与基站之间的最大延迟时间（s）。延迟通常是由于差分数据传输链路异常所导致的数据丢失。demo5版本代码可以将其从默认值提高，因为在首次 fix-and-hold 后，即使这个值变得较大，仍可以获得较好的结果。

### pos2-rejionno = <font style="color: #1560BD;">1.0-2.0</font>
`pos2-rejionno` 指的 **EKF 新息** 的拒绝阈值，这里新息指的是 **验前残差**，偶尔大家也会称其为 **OMC（Observation Minus Computed）** 以便于记忆。

如果新息大于该值（m），则拒绝该观测值。在 demo5 b33 代码之前，该值未经调整直接应用于伪距和载波相位。在较新版本中，该值仍直接应用于相位观测值，但伪距观测值会乘以 `eratio`。这使得可以根据相位测量设置合适的值。

demo5 作者通常将其设置为1.0，这有助于捕获并避免未标记的周跳，但有时需要将其设置得更高。设置得过低可能导致滤波器在数据质量较差时发散，因此 demo5 作者将默认值设置为 2.0，不过常用值则是 1.0。

## OUTPUT
### out-solformat = enu, llh, xyz
如果对流动站（rover）和基准站（base）之间的相对距离感兴趣，可以将out-solformat设置为“enu”。如果对绝对位置感兴趣，可以设置为“llh”，但请确保在“ant2”配置中准确设置基准站的位置。

如果需要准确的z轴测量，请谨慎使用此设置。只有“llh”格式才能在流动站处于恒定高度时提供恒定的z高度。“enu”和“xyz”是笛卡尔坐标系，因此z轴遵循一个平面，而不是地球的曲率。如果基准站距离流动站较远，这种设置可能会导致较大的误差，因为随着距离增加，地球曲率的影响会增大。

### out-outhead = on
对解的功能没有影响，只是向结果文件中输出更多信息。

### out-outopt = on
对解的功能没有影响，只是向结果文件中输出更多信息。

### out-outstat = residual
对解的功能没有影响，只是将残差输出到一个文件。残差对于调试解的问题非常有用，并且只要残差文件与解文件在同一文件夹中，就可以使用RTKPLOT进行绘图。

## STATISTICS
### stats-eratio1 = 300, stats-eratio2 = 300, stats-eratio5 = 300
伪距观测值与载波相位观测值的标准差比率。低成本接收机设置较大的值效果会更好（如 300），而对于更昂贵的接收器，默认值则设定为 100，因为它们的伪距观测噪声较小。较大的值往往会使卡尔曼滤波器更快收敛，也会加速首次固定，但同时也增加了假固定的可能性。如果增加该值，建议将pos2-arthres1设置得足够低，以防止滤波器尚未收敛时就进行固定。demo5作者认为增加此值的作用类似于在伪距平滑算法中增加时间常数，它会滤除伪距测量中更多的高频成分，从而保留低频成分。

### stats-prnaccelh = 3.0
如果启用了接收机动态（Rec Dynamics），则可以使用该值来设置流动站接收机水平分量加速度的标准差。该值应包括所有频率的加速度，而不仅仅是低频率加速度。它应反映流动站天线的任何运动，而不仅仅是整个流动站的运动，因此可能比你想象的要大。它将包括振动、道路颠簸等引起的加速度，以及整个流动站更明显的刚体加速度。可以通过将此值设置为一个较大的值来运行解算，然后使用RTKPLOT检查解算文件中加速度值进行估计。

### stats-prnaccelv = 1.0
水平加速度中的描述相较于垂直加速度分量甚至更为适用，因为在许多应用中，我们关注的加速度通常都集中在水平分量上。最好从实际的GPS测量数据中推导这个值，而不是基于对刚体流动站的预期。过高估计这些值比低估它们更好。

## POSITIONS
### ant2-postype = rinexhead, llh, single
这是基站天线的位置。如果你只关心基站与移动站之间的相对距离，那么这个位置值不需要特别精确。在后处理中，我通常使用RINEX文件头中的近似基站位置。尽管标注为近似值，但如果你使用的是来自CORS参考站的RINEX文件，这个位置通常会很精确。否则，如果我需要绝对位置，我会先将基站数据与附近的参考站进行处理，以获得精确位置，然后使用“llh”或“xyz”选项指定该位置。对于实时处理，由于通常不知道基站的确切位置，所以可以使用“single”选项，该选项利用数据的单点解来估算基站的大致位置。

### ant2-maxaveep = 1
指定当“postype”设置为“single”时，用于确定基站位置的平均样本数量。

demo5作者将其设置为1，以防止卡尔曼滤波器开始收敛后基站位置发生变化，因为这似乎会导致首次定位时间变长。在大多数后处理情况下，基站位置将来自RINEX文件头，因此你不会使用这个设置。然而，如果你正在处理RTCM文件，即使在后处理中，你可能也需要使用它。

## MISC
### misc-timeinterp =off,on
对基站观测数据进行时间插值。如果基站观测数据的采样时间间隔大于5秒，我通常会将其设置为“on”。
