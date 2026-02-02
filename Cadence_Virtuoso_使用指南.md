# RRAM模型在Cadence Virtuoso中的使用指南

本文档介绍如何在Cadence Virtuoso中使用RRAM Verilog-A模型来仿真I-V曲线。

---

## 一、文件说明

| 文件名 | 说明 |
|--------|------|
| `rram.va` | RRAM Verilog-A模型主文件 |
| `1T1R.sp` | 1T1R测试电路SPICE网表（参考用） |
| `65nm_bulk.pm` | PTM 65nm CMOS工艺模型 |

---

## 二、在Cadence Virtuoso中导入RRAM模型

### 方法一：使用Verilog-A编译器导入

1. **创建新的Library**
   - 打开Library Manager: `Tools → Library Manager`
   - 点击 `File → New → Library`
   - 输入Library名称（如：`RRAM_lib`），选择 `Attach to existing technology library` 或 `Don't need a techfile`

2. **创建新的CellView用于Verilog-A**
   - 在新建的Library中，`File → New → Cell View`
   - Cell Name: `rram`
   - View Name: `veriloga`
   - Tool: `Text Editor`

3. **复制Verilog-A代码**
   - 将 `rram.va` 的内容复制到编辑器中
   - 保存并关闭

4. **编译Verilog-A模型**
   - 在CIW (Command Interpreter Window)中执行：
   ```
   ahdlCompile("./rram.va")
   ```
   - 或者在Terminal中使用：
   ```bash
   ncvlog -work worklib -cdslib ./cds.lib -logfile ncvlog.log -errormax 15 -update -linedebug -status -ams rram.va
   ```

### 方法二：直接创建Symbol并关联

1. 编译完成后，为RRAM模型创建Symbol：
   - 选择 `rram` cell
   - `File → New → Cell View`
   - View Name: `symbol`
   - Tool: `Symbol Editor`

2. 创建symbol图形：
   - 绘制一个电阻器符号（或RRAM专用符号）
   - 添加两个pin：`Nt`（顶部端口）和 `Nb`（底部端口）
   - 保存symbol

---

## 三、创建I-V曲线仿真测试电路

### 3.1 简单的RRAM I-V测试电路

创建新的Schematic用于仿真：

```
         Vin (DC Sweep)
          |
         (+)
    ┌────┤ V1 ├────┐
    │    (-)       │
    │              │
    │     Nt       │
    │    ┌───┐     │
    └────┤   ├─────┘
         │RRAM│
    ┌────┤   ├─────┐
    │    └───┘     │
    │     Nb       │
    │              │
    │             GND
   GND
```

**电路组成：**
- V1: DC电压源，用于扫描电压
- RRAM: 你的RRAM模型实例
- 接地节点

### 3.2 Schematic设置步骤

1. **创建新的Schematic**
   - `File → New → Cell View`
   - Cell Name: `RRAM_IV_test`
   - View Name: `schematic`

2. **放置元件**
   - 放置RRAM symbol（从你的RRAM_lib中）
   - 放置DC电压源 `vdc` (从 analogLib)
   - 放置地节点 `gnd` (从 analogLib)

3. **设置RRAM实例参数**
   双击RRAM实例，设置参数：
   ```
   gap_ini = 1.7e-9    (初始gap距离，HRS状态)
   或
   gap_ini = 0.1e-9    (初始gap距离，LRS状态)
   tstep = 1e-9        (内部时间步长)
   ```

4. **连接电路**
   - V1的正端连接到RRAM的Nt端
   - V1的负端和RRAM的Nb端连接到地

---

## 四、仿真设置 - 获取I-V曲线

### 4.1 方法一：DC扫描仿真（静态I-V）

⚠️ **注意**：由于RRAM是忆阻器件，其行为依赖于历史状态，DC扫描可能无法完整展示SET/RESET迟滞特性。推荐使用瞬态仿真。

1. **启动ADE L**
   - `Launch → ADE L`

2. **设置DC分析**
   ```
   Analysis → Choose → dc
   - Sweep Variable: Component Parameter
   - Component Name: /V1
   - Parameter Name: dc
   - Sweep Range: Start=-2, Stop=2
   - Sweep Type: Linear, Step=0.01
   ```

3. **设置输出**
   ```
   Outputs → To Be Plotted → Select on Schematic
   - 选择RRAM两端的电流：I("/RRAM/Nt")
   ```

### 4.2 方法二：瞬态仿真（动态I-V，推荐）

这是获取完整迟滞I-V曲线的推荐方法：

1. **修改电压源为PWL或Pulse**
   - 使用三角波或正弦波电压源
   - 示例PWL设置：
   ```
   V1: vpwl (Piece-Wise Linear)
   
   时间点和电压值：
   0      0V
   10n    2V     (正向扫描到SET电压)
   20n    0V     (返回)
   30n   -2V     (负向扫描到RESET电压)
   40n    0V     (返回)
   ```

2. **设置瞬态分析**
   ```
   Analysis → Choose → tran
   - Stop Time: 40n
   - Accuracy Defaults: moderate 或 conservative
   - 勾选 errpreset
   ```

3. **设置输出**
   ```
   Outputs → To Be Plotted → Select on Schematic
   - 选择RRAM电流和电压
   ```

4. **绘制I-V曲线**
   仿真完成后，在波形窗口中：
   ```
   Tools → Calculator
   - 使用XY plot功能
   - X轴: V("/Nt") 或 V("/V1")
   - Y轴: I("/RRAM/Nt")
   ```
   
   或直接使用命令：
   ```
   plot(getData("I" ?result "tran") vs getData("V" ?result "tran"))
   ```

---

## 五、1T1R配置仿真（参考1T1R.sp）

如果需要仿真1T1R（1晶体管1RRAM）结构：

### 5.1 电路结构

```
       BL (Bit Line)
        │
       Nt
    ┌───┴───┐
    │ RRAM  │
    └───┬───┘
       Nb
        │
    ┌───┴───┐
    │       │
WL──┤ NMOS  │
    │ (M1)  │
    └───┬───┘
        │
       SL (Source Line)
```

### 5.2 操作模式

根据 `1T1R.sp` 文件中的设置：

| 操作 | WL电压 | BL电压 | SL电压 |
|------|--------|--------|--------|
| SET  | 1.0V   | 2.0V   | 0V     |
| RESET| 2.9V   | 0V     | 1.9V   |

### 5.3 仿真脉冲设置

```
SET脉冲:   脉宽=10ns, 起始时间=5ns
RESET脉冲: 脉宽=10ns, 起始时间=20ns
```

---

## 六、RRAM模型参数说明

### 关键可调参数

| 参数 | 默认值 | 单位 | 说明 |
|------|--------|------|------|
| `L` | 5e-9 | m | 氧化层厚度 |
| `gap_min` | 0.1e-9 | m | 最小gap距离（LRS） |
| `gap_max` | 1.7e-9 | m | 最大gap距离（HRS） |
| `gap_ini` | 0.1e-9 | m | 初始gap距离 |
| `Eag` | 1.501 | eV | 空位产生激活能 |
| `Ear` | 1.5 | eV | 空位复合激活能 |
| `I0` | 6.14e-5 | A | I-V特性电流系数 |
| `V0` | 0.43 | V | I-V特性电压系数 |
| `T0` | 298 | K | 环境温度 |
| `tstep` | 1e-9 | s | 最大内部时间步长 |

### 状态选择

- **低阻态 (LRS)**: 设置 `gap_ini = gap_min = 0.1e-9`
- **高阻态 (HRS)**: 设置 `gap_ini = gap_max = 1.7e-9`

---

## 七、仿真结果示例

成功仿真后，你应该能看到类似下图的I-V迟滞曲线：

- **SET过程** (蓝色): 正向电压扫描，电阻从HRS转变为LRS
- **RESET过程** (红色): 负向电压扫描，电阻从LRS转变为HRS

参考仓库中的 `model fitting.jpg` 图片查看预期的I-V曲线形状。

---

## 八、常见问题排解

### Q1: 编译Verilog-A时出错
**A**: 确保包含以下头文件在搜索路径中：
- `constants.vams`
- `disciplines.vams`

这些文件通常在Cadence安装目录的 `tools/dfII/etc/veriloga` 下。

### Q2: 仿真不收敛
**A**: 尝试以下方法：
1. 减小 `tstep` 参数（如改为 `1e-10`）
2. 使用更保守的仿真精度设置
3. 降低电压扫描速率
4. 在ADE中设置：`Options → Simulator → errpreset = conservative`

### Q3: I-V曲线没有迟滞特性
**A**: 
1. 确保使用瞬态仿真而非DC扫描
2. 检查 `gap_ini` 初始值设置
3. 确保电压幅度足够大（通常需要 >1.5V）

### Q4: 如何观察内部状态变量？
**A**: 模型输出以下内部状态：
- `V(gap_out)`: 当前gap距离
- `V(temp_out)`: 当前温度
- `V(R_out)`: 当前电阻值（在Vread下）

---

## 九、ADE L/XL 仿真脚本示例

```ocean
; OCEAN脚本示例
simulator('spectre)
design("./RRAM_lib" "RRAM_IV_test" "schematic")
resultsDir("./simulation")
modelFile('("./65nm_bulk.pm" ""))

analysis('tran ?stop "40n" ?errpreset "conservative")

desVar("Vamp" 2.0)

temp(27)
run()

selectResult('tran)
plot(getData("I" "/RRAM/Nt") vs getData("V" "/Nt"))
```

---

## 十、参考资料

- RRAM Verilog-A模型论文（请参考模型中的物理方程）
- Cadence Virtuoso用户手册
- Verilog-A语言参考手册

---

**如有问题，请参考仓库中的 `RRAM_VerilogA_manual.pdf` 文档获取更详细的模型说明。**
