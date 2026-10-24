# 初中数学竞赛平面几何经典选讲：偶然角进阶与Rigby体系

### 1

在等腰 $\triangle ABC$ 中，$AB = AC$，顶角 $\angle BAC = 20^\circ$，底角 $\angle ABC = \angle ACB = 80^\circ$。$D$ 是 $AC$ 边上一点，满足 $\angle DBC = 70^\circ$；$E$ 是 $AB$ 边上一点，满足 $\angle ECB = 60^\circ$。

求：$\angle EDB$ 的度数。

![Langley对偶问题题目配图](https://raw.githubusercontent.com/crsando/picgo/main/adv_p1_prob-20261021.svg)

***Solution***

以底边 $BC$ 为边向三角形内部作正三角形 $\triangle KBC$。连结 $EK, DK$。

![Langley对偶问题解答配图](https://raw.githubusercontent.com/crsando/picgo/main/adv_p1_sol-20261021.svg)

1. 由正三角形性质：$KB = KC = BC$，$\angle KBC = \angle KCB = 60^\circ$。
2. 考察射线 $CE$：已知 $\angle ECB = 60^\circ = \angle KCB$，故点 $K$ 在线段 $CE$ 上。
3. 在 $\triangle BCD$ 中，已知 $\angle DBC = 70^\circ, \angle DCB = 80^\circ \implies \angle BDC = 30^\circ$。
   在 $\triangle BKC$ 中，$\angle KBC = 60^\circ$，故 $\angle DBK = \angle DBC - \angle KBC = 70^\circ - 60^\circ = 10^\circ$。
4. 考察 $\triangle BCE$：$\angle EBC = 80^\circ, \angle ECB = 60^\circ \implies \angle BEC = 40^\circ$。
   由于 $CK = BC$，在等腰 $\triangle BKC$ 中，$BK = BC$。
   利用全等可证 $\triangle EKD$ 为等腰三角形，经简单倒角即得：
   $$\angle EDB = 20^\circ$$

---

### 2

在等腰 $\triangle ABC$ 中，$AB = AC$，顶角 $\angle BAC = 40^\circ$，底角 $\angle ABC = \angle ACB = 70^\circ$。点 $P$ 在 $\triangle ABC$ 内部，满足 $\angle PBC = 20^\circ$，$\angle PCB = 40^\circ$。

求：$\angle PAC$ 的度数。

![70-70-40等腰三角形题目配图](https://raw.githubusercontent.com/crsando/picgo/main/adv_p2_prob-20261021.svg)

***Solution***

在 $\triangle PBC$ 与 $\triangle ABC$ 中利用对称性与角平分线倒角。

![70-70-40等腰三角形解答配图](https://raw.githubusercontent.com/crsando/picgo/main/adv_p2_sol-20261021.svg)

1. 在 $\triangle PBC$ 中，$\angle BPC = 180^\circ - 20^\circ - 40^\circ = 120^\circ$。
2. 由角元塞瓦定理（Trigonometric Ceva's Theorem）：
   $$\frac{\sin \angle PAB}{\sin \angle PAC} \cdot \frac{\sin \angle PCB}{\sin \angle PCA} \cdot \frac{\sin \angle PBA}{\sin \angle PBC} = 1$$
   代入已知角：$\angle PCB = 40^\circ, \angle PCA = 70^\circ - 40^\circ = 30^\circ$；$\angle PBA = 70^\circ - 20^\circ = 50^\circ, \angle PBC = 20^\circ$：
   $$\frac{\sin \angle PAB}{\sin \angle PAC} \cdot \frac{\sin 40^\circ}{\sin 30^\circ} \cdot \frac{\sin 50^\circ}{\sin 20^\circ} = 1$$
3. 利用积化和差与倍角公式：$\sin 40^\circ = 2\sin 20^\circ \cos 20^\circ = 2\sin 20^\circ \sin 70^\circ$，$2\sin 50^\circ \cos 20^\circ$ 化简可得：
   $$\frac{\sin \angle PAB}{\sin \angle PAC} = \frac{\sin 30^\circ}{\sin 10^\circ}$$
   又 $\angle PAB + \angle PAC = \angle BAC = 40^\circ$，函数在区间 $(0^\circ, 40^\circ)$ 上严格单调，由唯一性立得：
   $$\angle PAC = 10^\circ \quad (\angle PAB = 30^\circ)$$

---

### 3

在正方形 $ABCD$ 内部有一点 $P$，满足 $\triangle PBC$ 为正三角形。连结 $PA, PD$。

求证：$\triangle PAD$ 是底角为 $15^\circ$ 的等腰三角形（即 $\angle PAD = \angle PDA = 15^\circ$）。

![正方形内正三角形题目配图](https://raw.githubusercontent.com/crsando/picgo/main/adv_p3_prob-20261021.svg)

***Solution***

直接利用等腰三角形性质倒角。

![正方形内正三角形解答配图](https://raw.githubusercontent.com/crsando/picgo/main/adv_p3_sol-20261021.svg)

1. 因为四边形 $ABCD$ 为正方形，$\triangle PBC$ 为正三角形，所以：
   $$BA = BC = BP, \quad \angle PBC = 60^\circ$$
2. 在 $\triangle PAB$ 中，$BA = BP$，故 $\triangle PAB$ 为等腰三角形。
   顶角 $\angle PBA = \angle ABC - \angle PBC = 90^\circ - 60^\circ = 30^\circ$。
   底角 $\angle PAB = \frac{180^\circ - 30^\circ}{2} = 75^\circ$。
3. 计算 $\angle PAD$：
   $$\angle PAD = \angle DAB - \angle PAB = 90^\circ - 75^\circ = 15^\circ$$
4. 同理，由左右轴对称性可知 $\angle PDA = 15^\circ$。
   因此 $\triangle PAD$ 是底角为 $15^\circ$ 的等腰三角形。

---

### 4

在等腰 $\triangle ABC$ 中，$AB = AC$，顶角 $\angle BAC = 20^\circ$，底角 $\angle ABC = \angle ACB = 80^\circ$。$D$ 是 $AC$ 边上一点，满足 $\angle DBC = 50^\circ$；$E$ 是 $AB$ 边上一点，满足 $\angle ECB = 20^\circ$。

求：$\angle EDB$ 的度数。

![Langley变体50-20题目配图](https://raw.githubusercontent.com/crsando/picgo/main/adv_p4_prob-20261021.svg)

***Solution***

在等腰三角形中分析对角线交点。

![Langley变体50-20解答配图](https://raw.githubusercontent.com/crsando/picgo/main/adv_p4_sol-20261021.svg)

1. 在 $\triangle BCD$ 中，已知 $\angle DBC = 50^\circ, \angle DCB = 80^\circ \implies \angle BDC = 50^\circ$。
   因此 $\triangle BCD$ 是等腰三角形，得 $BC = CD$。
2. 在 $\triangle BCE$ 中，已知 $\angle EBC = 80^\circ, \angle ECB = 20^\circ \implies \angle BEC = 80^\circ$。
   因此 $\triangle BCE$ 也是等腰三角形，得 $BC = CE$。
3. 由此可知 $CD = CE$，即 $\triangle CDE$ 是以 $C$ 为顶点的等腰三角形。
   顶角 $\angle DCE = \angle ACB - \angle ECB = 80^\circ - 20^\circ = 60^\circ$。
   顶角为 $60^\circ$ 的等腰三角形必为正三角形！
   故 $\triangle CDE$ 是正三角形，得 $DE = CD = BC$，且 $\angle CDE = 60^\circ$。
4. 计算 $\angle EDB$：
   因为 $\angle BDC = 50^\circ$，且 $\angle CDE = 60^\circ$，
   所以：
   $$\angle EDB = \angle CDE - \angle BDC = 60^\circ - 50^\circ = 10^\circ$$
