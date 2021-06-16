---
title: 时钟验证（Tick Verification）
---

这样可以设计插槽中的刻度线的标准和验证。 它还描述了错误处理和罚没条件，包括系统如何处理不满足这些要求的传输。

# 插槽结构

每个插槽必须包含一个预期的`ticks_per_slot`滴答次数。 插槽中的最后一个碎片必须仅包含最后一次滴答，而不能包含其他任何内容。 领导者还必须用`LAST_SHRED_IN_SLOT`标志标记包含最后一次滴答的碎片。 在滴答之间，必须有`hashes_per_tick`个哈希值。

# 处理不良传播

恶意传输`T`的处理方式有两种：

1. 如果领导者可以为同一个插槽生成一些错误的传输`T`以及一些替代传输`T‘`，而不会违反重复传输的任何罚没规则(例如，如果`T‘`是`T`的子集)，则群集必须处理两种传输均处于活动状态的可能性。

因此，这意味着我们无法将错误的传输`T`标记为已死，因为集群可能已就`T‘`达成共识。 这些案件需要严厉的证据来惩治这种不良行为。

2. 否则，我们可以简单地将插槽标记为已死且无法播放。 根据可行性，罚没可能是必要的，也可能不是必要的。

# Blockstore 接收碎片

当 blockstore 收到一个新的碎片`s`，有两个实例:

1. 将`s`标记为`LAST_SHRED_IN_SLOT`，然后检查该存储区中是否存在用于此插槽的碎片`s‘`，其中`s'.index > s.index`。如果是，则将`s`和`s‘`一起构成罚没的证明。

2. Blockstore已经收到一个标记为`LAST_SHRED_IN_SLOT`的带有索引`i`的碎片`s'`。 如果`s.index > i`，则`s`和`s'`一起构成一个罚没证明。 在这种情况下，blockstore也不会插入`s`。

3. 相同索引的重复碎片将被忽略。 相同索引的不可重复的碎片是一个可罚没的条件。 有关这种情况的详细信息，请参见`领导者重复区块罚没`章节。

# 重播和验证滴答

1. 重播阶段的重播来自blockstore的条目，它跟踪每个插槽中已观察到的滴答数量，并确认在各个标记之间有`hashes_per_tick`个哈希值。 在播放完最后一个碎片的滴答之后，重播阶段会检查滴答的总数。

失败场景1：如果有两个连续的滴答，其间的哈希数为`！=Hashes_per_tick`，则将此插槽标记为无效。

失败情况2：如果滴答数量等于 != `ticks_per_slot`，则将插槽标记为无效。

失败情况3：如果滴答的数量达到`ticks_per_slot`，但我们仍未看到`LAST_SHRED_IN_SLOT`，则将此插槽标记为无效。

2. 当ReplayStage达到标记为最后一个碎片时，它将检查这个碎片是否为一次滴答。

失败情况：如果带有`LAST_SHRED_IN_SLOT`标志的带符号碎片无法反序列化为滴答(无法反序列化或反序列化为条目)，则将此插槽标记为无效。