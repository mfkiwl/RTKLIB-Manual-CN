# 10. 伪距单点定位

<img src="https://raw.githubusercontent.com/salmoshu/Winchell-ImgBed/main/img/20250708-174707.jpg"/>
<p style="text-align: center; font-family: 'Microsoft YaHei', SimSun, Arial, sans-serif; font-size: 14px;">图10-1 伪距单点定位流程</p>

RTKLIB 伪距单点定位源码的解析参考了资料[1][2][3]，对于`satposs()`函数的源码分析可以参考第7章。

## 10.1 核心定位函数

### 10.1.1 pntpos()：单点定位入口函数

**1. 参数列表**

```c
/* args */
obsd_t   *obs     I   observation data
int       n       I   number of observation data
nav_t    *nav     I   navigation data
prcopt_t *opt     I   processing options
sol_t    *sol     IO  solution
double   *azel    IO  azimuth/elevation angle (rad) (NULL: no output)
ssat_t   *ssat    IO  satellite status              (NULL: no output)
char     *msg     O   error message for error exit
/* return */
int                   status (1:ok,0:error)
```

**2. 执行流程**

- `sol->time`赋值第一个观测值（顺序存储的第一颗卫星）的时间 
- 如果处理选项不是 SPP（针对 PPP），电离层校正选 Klobuchar 模型 ，对流层校正采用 Saastmoinen 模型 
- 调用`satposs()`计算卫星位置、速度和钟差（ECEF 坐标系） 
    - `rs [(0:2)+i*6]`：obs[i] 卫星位置 {x, y, z} (m) 
    - `rs [(3:5)+i*6]`：obs[i] 卫星速度 {vx, vy, vz} (m/s) 
    - `dts[(0:1)+i*2]`：obs[i] 卫星钟差与钟漂 {bias, drift} (s, s/s) 
    - `var[i]`：卫星的误差方差 (m^2)，代表了卫星位置和钟差所包含的误差水平，最终会合入伪距的方差中
    - `svh[i]`：卫星健康标志 (-1:correction not available) 
- 调用`estpos()`使用伪距进行位置估计，采用加权最小二乘，其中会调用 `valsol` 进行卡方检验和 GDOP 检验 
- 若`estpos()`中`valsol`检验失败，即位置估计失败，会调用`raim_fde`接收机自主完好性监测重新估计，前提是卫星数 > 6、对应参数解算设置`opt->posopt[4]=1`
- 调用`estvel()`使用多普勒进行速度估计
- 保存方位角和俯仰角信息，赋值卫星状态结构体ssat

**3. 注意事项**

- **关于钟漂信息**：这里只计算了接收机的钟差，而没有计算接收机的频漂，原因在于 estvel 函数中虽然计算得到了接收机频漂，但并没有将其输出到 sol_t:dtr中。
- **C语言内存使用问题**：C语言中用 `malloc` 申请的内存需要自己调用 `free` 来予以回收，源码中的 `mat` `、imat` `、zeros` 等函数都只是申请了内存，并没有进行内存的回收，在使用这些函数时，用户必须自己调用 `free` 来回收内存！源码中将使用这些函数的代码放置在同一行，在调用函数结尾处也统一进行内存回收，位置较为明显，不致于轻易忘记。

**4. 问题思考**
- **RTKLIB观测数据中的时间**：源码中将 obs[0].time作为星历选择时间传递给 satposs函数，这样对于每一颗观测卫星，都要使用第一颗观测卫星的数据接收时间作为选择星历的时间标准。是否应该每颗卫星都使用自己的观测时间？或者应该使用每颗卫星自己的信号发射时间？还是说这点差别对选择合适的星历其实没有关系？**该问题的分析参考附录B.1**。
- **raim_fde 对卫星数目的要求**：这里规定能够执行 raim_fde 函数的前提是数目大于等于 6，感觉不是只要大于等于 5就可以了吗？**该问题的分析参考附录B.2**。

::: details 点击查看代码
```c
extern int pntpos(const obsd_t *obs, int n, const nav_t *nav,
                  const prcopt_t *opt, sol_t *sol, double *azel, ssat_t *ssat,
                  char *msg)
{
    prcopt_t opt_=*opt;
    double *rs,*dts,*var,*azel_,*resp;
    int i,stat,vsat[MAXOBS]={0},svh[MAXOBS];
    
    trace(3,"pntpos  : tobs=%s n=%d\n",time_str(obs[0].time,3),n);
    sol->stat=SOLQ_NONE;
    
    if (n<=0) {
        strcpy(msg,"no observation data");
        return 0;
    }

    sol->time=obs[0].time; // sol->time赋值第一个观测值的时间
    msg[0]='\0';
    
    rs=mat(6,n); dts=mat(2,n); var=mat(1,n); azel_=zeros(2,n); resp=mat(1,n);
    
    if (opt_.mode!=PMODE_SINGLE) { /* for precise positioning */
        opt_.ionoopt=IONOOPT_BRDC; // 电离层校正选 Klobuchar 广播星历模型
        opt_.tropopt=TROPOPT_SAAS; // 对流层校正采用 Saastmoinen 模型
    }
    
    // 计算卫星位置、速度和钟差
    /* satellite positons, velocities and clocks */
    satposs(sol->time,obs,n,nav,opt_.sateph,rs,dts,var,svh);
    
    // 用伪距位置估计，加权最小二乘，其中会调用 valsol 进行卡方检验和 GDOP 检验
    /* estimate receiver position with pseudorange */
    stat=estpos(obs,n,rs,dts,var,svh,nav,&opt_,sol,azel_,vsat,resp,msg);
    
    /* RAIM FDE */
    if (!stat&&n>=6&&opt->posopt[4]) {  
        // estpos 中 valsol 检验失败，即位置估计失败，则会调用 RAIM 接收机自主完好性监测重新估计，
        // 前提是卫星数 > 6，且对应参数解算设置 opt->posopt[4]=1，即pos1-posopt5=on（RAIM FDE）
        stat=raim_fde(obs,n,rs,dts,var,svh,nav,&opt_,sol,azel_,vsat,resp,msg);
    }
    // 调用 estvel 进行多普勒速度估计
    /* estimate receiver velocity with Doppler */
    if (stat) {
        estvel(obs,n,rs,dts,nav,&opt_,sol,azel_,vsat);
    }
    if (azel) {
        for (i=0;i<n*2;i++) azel[i]=azel_[i];
    }
    if (ssat) { // 保存卫星状态信息至ssat结构体
        for (i=0;i<MAXSAT;i++) {
            ssat[i].vs=0;
            ssat[i].azel[0]=ssat[i].azel[1]=0.0;
            ssat[i].resp[0]=ssat[i].resc[0]=0.0;
            ssat[i].snr[0]=0;
        }
        for (i=0;i<n;i++) {
            ssat[obs[i].sat-1].azel[0]=azel_[  i*2];
            ssat[obs[i].sat-1].azel[1]=azel_[1+i*2];
            ssat[obs[i].sat-1].snr[0]=obs[i].SNR[0];
            if (!vsat[i]) continue;
            ssat[obs[i].sat-1].vs=1;
            ssat[obs[i].sat-1].resp[0]=resp[i];
        }
    }
    free(rs); free(dts); free(var); free(azel_); free(resp);
    return stat;
}
```
:::

### 10.1.2 estpos()：通过伪距实现绝对定位

计算出接收机的位置和钟差，并返回每颗卫星的{方位角、仰角}、定位时有效性、定位后伪距残差。 

**1. 参数列表**

```c
/* args */
obsd_t   *obs      I   观测量数据
int      n         I   观测量数据的数量
double   *rs       I   卫星位置和速度，长度为6*n，{x,y,z,vx,vy,vz}(ecef)(m,m/s)
double   *vare     I   卫星位置和钟差的协方差 (m^2)
int      *svh      I   卫星健康标志 (-1:correction not available)
nav_t    *nav      I   导航数据
prcopt_t *opt      I   处理过程选项
prcopt_t *opt      I   处理过程选项
sol_t    *sol      IO  solution
double   *azel     IO  方位角和俯仰角 (rad)
int      *vsat     IO  卫星在定位时是否有效
double   *resp     IO  定位后伪距残差 (P-(r+c*dtr-c*dts+I+T))
char     *msg      O   错误消息
/* return */
int                    status (1:ok,0:error)
```

**2. 执行流程**

* 赋值`x[i]`：将 `sol->rr` 的前 3 项赋值给 `x` 数组（如果是第一次定位，那么 `x` 初值为 0）。
* 开始迭代计算：迭代次数`MAXITR`默认为10。
  - 调用`rescode()`：计算当前迭代的伪距残差 `v`、设计矩阵 `H`、伪距残差的方差 `var`（用以加权）、所有观测卫星的方位角和仰角 `azel`，定位时有效性 `vsat`、定位后伪距残差 `resp`、参与定位的卫星个数 `ns` 和方程个数 `nv`。
  - 确定方程组中方程的个数`nv`要大于未知数`NX`的个数。
  - 以伪距残差的标准差的倒数 `1/sqrt(var)` 作为权重，对`H`和`v`分别左乘权重对角阵，得到加权之后的`H`和`v`。
  - 调用`lsq()`最小二乘函数,得到当前`x`的改正数`dx`和定位误差协方差矩阵。
  - 将`lsq()`中求得的 `dx` 加入到当前`x`值中，得到更新之后的`x`值 
  - 如果求得的修改量`dx`小于截断因子(目前是1E-4)：
    * 则将`x`作为最终的定位结果，对`sol`的相应参数赋值。
    * 之后再调用对定位结果进行卡方检验和 GDOP 检验，检验当前结果是否符合要求（伪距残余小于某个卡方值以及 GDOP 小于某个门限值）。
    * 大于截断因子，则进行下一次循环。
  - 如果超过了规定的循环次数，则输出发散信息后，`return 0`。

**3. 注意事项**

- **关于加权最小二乘**：这里的权重值是对角阵，这是建立在假设不同测量值的误差之间是彼此独立的；另外，这个权重值并不单是伪距测量误差的，而是方程右端 b 整体的测量误差。最后，大部分资料上这里都是把权重矩阵 W 保留到方程的解的表达式当中，而这里是直接对 H 和 v 分别左乘权重对角阵，得到加权之后的 H 和 v ，这样做的等价的，其好处是减少运算量。
- **校验函数的使用**：如果某次迭代过程中步长小于门限值(1e-4)，但经 valsol 函数检验后该解无效，则会直接返回 0，并不会再进行下一次迭代计算。实际上卡方校验的条件过于严格，运用到低成本接收机时可以直接关闭卡方校验。
- **sol中的钟差修正**：在对 sol 结构体赋值时，sol 中实际存储的是减去接收机钟差后的信号观测时间。
- **估计状态的维度**：源码中定位方程的个数 nv 要大于有效观测卫星的个数 ns，这里为了防止亏秩，并且又加了 3 个未知数和观测方程（对应了其他系统的钟差项）。
- **rescode的处理**：在每一次重新调用 rescode函数时，其内部并没有对 v 、H 和 var 进行清零处理，所以当方程数变少时，可能会存在尾部仍保留上一次数据的情况，但是因为数组相乘时都包含所需计算的长度 nv，所以这种情况并不会对计算结果造成影响。

::: details 点击查看代码
```c
static int estpos(const obsd_t *obs, int n, const double *rs, const double *dts,
                  const double *vare, const int *svh, const nav_t *nav,
                  const prcopt_t *opt, sol_t *sol, double *azel, int *vsat,
                  double *resp, char *msg)
{
    double x[NX]={0},dx[NX],Q[NX*NX],*v,*H,*var,sig;
    int i,j,k,info,stat,nv,ns;
    
    trace(3,"estpos  : n=%d\n",n);
    
    v=mat(n+4,1); H=mat(NX,n+4); var=mat(n+4,1);
    
    // 赋值x[i]：将 sol->rr 的前 3 项赋值给 x 数组（如果是第一次定位，那么 x 初值为 0）
    for (i=0;i<3;i++) x[i]=sol->rr[i]; 

    for (i=0;i<MAXITR;i++) {
        // 首先调用 rescode 函数，计算当前迭代的伪距残差 v、几何矩阵 H
        // 伪距残差的方差 var、所有观测卫星的方位角和仰角 azel、定位时有效性 vsat、
        // 定位后伪距残差 resp、参与定位的卫星个数 ns 和方程个数 nv
        /* pseudorange residuals (m) */
        nv=rescode(i,obs,n,rs,dts,vare,svh,nav,x,opt,v,H,var,azel,vsat,resp,
                  &ns);
        
        if (nv<NX) { // 确定方程组中方程的个数要大于未知数的个数
            sprintf(msg,"lack of valid sats ns=%d",nv);
            break;
        }
        /* weighted by Std */ // 以伪距残差的标准差的倒数作为权重，对 H 和 v 分别左乘权重对角阵，得到加权之后的H和v
        for (j=0;j<nv;j++) {
            sig=sqrt(var[j]); // 这里的权重值是对角阵，这是建立在假设不同测量值的误差之间是彼此独立的基础上的
            // 直接对 H 和 v 分别左乘权重对角阵，得到加权之后的 H 和 v
            v[j]/=sig;  
            for (k=0;k<NX;k++) H[k+j*NX]/=sig;
        }
        // 调用 lsq 函数,得到当前 x 的修改量 dx 和定位误差协方差矩阵中的权系数阵 Q
        /* least square estimation */
        if ((info=lsq(H,v,NX,nv,dx,Q))) {
            sprintf(msg,"lsq error info=%d",info);
            break;
        }
        // 将lsq中求得的 dx 加入到当前 x 值中，得到更新之后的 x 值
        for (j=0;j<NX;j++) {
            x[j]+=dx[j];
        }
        // 如果求得的修改量dx小于截断因子(目前是1E-4)，则将x[j]作为最终的定位结果，
        // 对 sol 的相应参数赋值,之后再调用 valsol 函数确认当前解是否符合要求,参考 RTKLIB Manual P162
        // 否则，进行下一次循环。
        if (norm(dx,NX)<1E-4) {
            sol->type=0;
            // 解方程时的 dtr 单位是 m，是乘以了光速之后的，解出结果后赋给 sol->dtr 时再除以光速
            // sol->time 中存储的是减去接收机钟差后的信号观测时间
            sol->time=timeadd(obs[0].time,-x[3]/CLIGHT);
            sol->dtr[0]=x[3]/CLIGHT; /* receiver clock bias (s) */
            sol->dtr[1]=x[4]/CLIGHT; /* GLO-GPS time offset (s) */
            sol->dtr[2]=x[5]/CLIGHT; /* GAL-GPS time offset (s) */
            sol->dtr[3]=x[6]/CLIGHT; /* BDS-GPS time offset (s) */
            sol->dtr[4]=x[7]/CLIGHT; /* IRN-GPS time offset (s) */
            for (j=0;j<6;j++) sol->rr[j]=j<3?x[j]:0.0;
            for (j=0;j<3;j++) sol->qr[j]=(float)Q[j+j*NX];
            sol->qr[3]=(float)Q[1];    /* cov xy */
            sol->qr[4]=(float)Q[2+NX]; /* cov yz */
            sol->qr[5]=(float)Q[2];    /* cov zx */
            sol->ns=(uint8_t)ns;
            sol->age=sol->ratio=0.0;
            
            // 对定位结果进行卡方检验和GDOP检验
            /* validate solution */
            if ((stat=valsol(azel,vsat,n,opt,v,nv,NX,msg))) {
                sol->stat=opt->sateph==EPHOPT_SBAS?SOLQ_SBAS:SOLQ_SINGLE;
            }
            free(v); free(H); free(var);
            return stat;
        }
    }
    // 如果超过了规定的循环次数，则输出发散信息后，return 0
    if (i>=MAXITR) sprintf(msg,"iteration divergent i=%d",i);   
    
    free(v); free(H); free(var);
    return 0;
}
```
:::

### 10.1.3 estvel()：通过多普勒计算接收机速度

**1. 参数列表**

```c
/* args */
obsd_t   *obs      I   OBS观测数据
int       n        I   观测数据的数量
double   *rs       I   卫星位置和速度，长度为6*n，{x,y,z,vx,vy,vz}(ecef)(m,m/s)
double   *dts      I   卫星钟差，长度为2*n， {bias,drift} (s|s/s)
nav_t    *nav      I   导航数据
prcopt_t *opt      I   处理过程选项
sol_t    *sol      IO  solution
double   *azel     IO  方位角和俯仰角 (rad)
int      *vsat     IO  定位时有效卫星
char     *msg      O   错误消息
/* return */
int                    status (1:ok,0:error)
```

**2. 执行流程**

* 定速的初始值直接给定为 0 ，而不像定位时初值选上一历元的的位置（一定程度上因为 SPP 的速度估计精度较低）。
* for循环迭代计算，最大迭代次数（默认10次）
  * 调用`resdop()`，计算定速方程组左边的几何矩阵和右端的速度残差，返回定速时所使用的卫星数目
  * 调用最小二乘法`lsq()`函数，解出{速度、频漂}的改正量`dx`，累加到`x`中。 
  * 检查当前计算出的改正量的绝对值是否小于 1E-6 ，满足条件则说明当前解已经很接近真实值了，将接收机三个方向上的速度、协方差存入到 sol->rr 中 。否则进行下一次循环。

**3. 注意事项**

- **钟漂信息**：最终向 sol 速度结果时，并没有存储所计算出的接收器钟漂。

::: details 点击查看代码
```c
static void estvel(const obsd_t *obs, int n, const double *rs, const double *dts,
                   const nav_t *nav, const prcopt_t *opt, sol_t *sol,
                   const double *azel, const int *vsat)
{
   // 定速的初始值直接给定为 0 ，而不像定位时初值选上一历元的的位置（一定程度上因为 SPP 的速度估计精度较低）
   double x[4]={0},dx[4],Q[16],*v,*H;
   double err=opt->err[4]; /* Doppler error (Hz) */
   int i,j,nv;
   
   trace(3,"estvel  : n=%d\n",n);
   
   v=mat(n,1); H=mat(4,n);
   
   for (i=0;i<MAXITR;i++) {
       // 调用 resdop，计算定速方程组左边的几何矩阵和右端的速度残余，返回定速时所使用的卫星数目
       /* range rate residuals (m/s) */
       if ((nv=resdop(obs,n,rs,dts,nav,sol->rr,x,azel,vsat,err,v,H))<4) {
           break;
       }
       // 调用最小二乘法 lsq 函数，解出{速度、频漂}的改正量 dx，累加到 x 中。
       // 频漂计算了，但并没有存储下来
       /* least square estimation */
       if (lsq(H,v,4,nv,dx,Q)) break;        
       for (j=0;j<4;j++) x[j]+=dx[j];
       // 检查当前计算出的改正量的绝对值是否小于 1E-6,
       // 是：则说明当前解已经很接近真实值了，将接收机三个方向上的速度存入到 sol->rr 中
       // 否：进行下一次循环
       if (norm(dx,4)<1E-6) {
           matcpy(sol->rr+3,x,3,1);
           sol->qv[0]=(float)Q[0];  /* xx */
           sol->qv[1]=(float)Q[5];  /* yy */
           sol->qv[2]=(float)Q[10]; /* zz */
           sol->qv[3]=(float)Q[1];  /* xy */
           sol->qv[4]=(float)Q[6];  /* yz */
           sol->qv[5]=(float)Q[2];  /* zx */
           break;
       }   
   }
   free(v); free(H);
}
```
:::

### 10.1.4 raim_fde()：接收机自主完好性检测与故障检测排除

通过每次排除一颗卫星计算，选取残差最小的一次作为最终解算结果，此时对应的卫星就是故障卫星。

**1. 参数列表**

```c
/* args */
const obsd_t   *obs     I    OBS观测数据
int             n       I    观测数据的数量
const double   *rs      I    卫星位置和速度，长度为6*n，{x,y,z,vx,vy,vz}(ecef)(m,m/s)
const double   *dts     I    卫星钟差，长度为2*n， {bias,drift} (s|s/s)
const double   *vare    I    卫星位置和钟差的协方差 (m^2)
const int      *svh     I    卫星健康标志 (-1:correction not available)
const nav_t    *nav     I    导航数据
const prcopt_t *opt     I    处理过程选项
sol_t          *sol     IO   解算结果
double         *azel    IO   方位角和俯仰角 (rad)
int            *vsat    IO   表征卫星在定位时是否有效
double         *resp    IO   观测卫星的伪距残差，(P-(r+c*dtr-c*dts+I+T)) (1*n)
char           *msg     O    错误信息
/* return */
int                          status (1:ok,0:error)
```

**2. 执行流程**

- 遍历所有卫星，每次舍弃一颗卫星，然后利用余下卫星进行定位解算。
- 累加使用当前卫星实现定位后的伪距残差平方和与可用卫星数目，如果 `nvsat<5` ，则说明当前卫星数目过少，无法进行 RAIM_FDE 操作。
- 计算伪距残差平方和的标准平均值，如果大于` rms`，则说明当前定位结果更合理，将 `stat` 置为 1，重新更新 `sol`、`azel`、`vsat`(当前被舍弃的卫星，此值置为0)、`resp` 等值，并将当前的 `rms_e` 更新到 `rms` 中（用以下一轮循环的判断）。
- 继续弃用下一颗卫星，重复上述操作。总而言之，在弃用一颗卫星条件下，选取伪距残差标准平均值最小的组合作为最终的结果输出。
- 如果 stat不为 0，则说明在弃用卫星的前提下有更好的解出现，输出信息，指出弃用了哪科卫星。

**3. 注意事项**

- 源码中有很多关于 `i`、`j`、`k` 的循环。其中，`i` 表示最外面的大循环，每次将将第 `i` 颗卫星舍弃不用，这是通过 `if (j==i) continue` 实现的；`j` 表示剩余使用的卫星的循环，每次进行相应数据的赋值；`k` 表示参与定位的卫星的循环，与 `j` 一起使用。

::: details 点击查看代码
```c
static int raim_fde(const obsd_t *obs, int n, const double *rs,
                    const double *dts, const double *vare, const int *svh,
                    const nav_t *nav, const prcopt_t *opt, sol_t *sol,
                    double *azel, int *vsat, double *resp, char *msg)
{
    obsd_t *obs_e;
    sol_t sol_e={{0}};
    char tstr[32],name[16],msg_e[128];
    double *rs_e,*dts_e,*vare_e,*azel_e,*resp_e,rms_e,rms=100.0;
    int i,j,k,nvsat,stat=0,*svh_e,*vsat_e,sat=0;
    
    trace(3,"raim_fde: %s n=%2d\n",time_str(obs[0].time,0),n);
    
    if (!(obs_e=(obsd_t *)malloc(sizeof(obsd_t)*n))) return 0;
    rs_e = mat(6,n); dts_e = mat(2,n); vare_e=mat(1,n); azel_e=zeros(2,n);
    svh_e=imat(1,n); vsat_e=imat(1,n); resp_e=mat(1,n); 

    // 遍历所有卫星，每次舍弃一颗卫星，然后利用余下卫星进行定位解算
    for (i=0;i<n;i++) {
        /* satellite exclution */
        for (j=k=0;j<n;j++) {
            if (j==i) continue;
            obs_e[k]=obs[j];
            matcpy(rs_e +6*k,rs +6*j,6,1);
            matcpy(dts_e+2*k,dts+2*j,2,1);
            vare_e[k]=vare[j];
            svh_e[k++]=svh[j];
        }
        // 调用 estpos 函数计算剩余卫星的定位结果
        /* estimate receiver position without a satellite */
        if (!estpos(obs_e,n-1,rs_e,dts_e,vare_e,svh_e,nav,opt,&sol_e,azel_e,
                    vsat_e,resp_e,msg_e)) {
            trace(3,"raim_fde: exsat=%2d (%s)\n",obs[i].sat,msg);
            continue;
        }
        // 累加计算当前卫星实现定位后的伪距残差平方和与可用卫星数目
        for (j=nvsat=0,rms_e=0.0;j<n-1;j++) {
            if (!vsat_e[j]) continue;
            rms_e+=SQR(resp_e[j]);
            nvsat++;
        }
        if (nvsat<5) { // 如果 nvsat<5，则说明当前卫星数目过少，无法进行 RAIM_FDE 操作
            trace(3,"raim_fde: exsat=%2d lack of satellites nvsat=%2d\n",
                  obs[i].sat,nvsat);
            continue;
        }
        rms_e=sqrt(rms_e/nvsat); // 计算伪距残差平方和的标准差rms_e
       
        trace(3,"raim_fde: exsat=%2d rms=%8.3f\n",obs[i].sat,rms_e);
       
        if (rms_e>rms) continue; // 如果伪距残差平方和的标准差rms_e>rms,继续下一次循环
       
        // 如果小于 rms，则说明当前定位结果更合理，将 stat 置为 1，
        // 重新更新 sol、azel、vsat(当前被舍弃的卫星，此值置为0)、resp 等值，并将当前的 rms_e 更新到 rms 中。
        /* save result */
        for (j=k=0;j<n;j++) {
            if (j==i) continue;
            matcpy(azel+2*j,azel_e+2*k,2,1);
            vsat[j]=vsat_e[k];
            resp[j]=resp_e[k++];
        }
        stat=1;
        *sol=sol_e;
        sat=obs[i].sat;
        rms=rms_e;
        vsat[i]=0;
        strcpy(msg,msg_e);
    }
    // 如果 stat不为 0，则说明在弃用卫星的前提下有更好的解出现，输出信息，指出弃用了哪颗卫星。
    if (stat) { 
        time2str(obs[0].time,tstr,2); satno2id(sat,name);
        trace(2,"%s: %s excluded by raim\n",tstr+11,name);
    }
    free(obs_e);
    free(rs_e ); free(dts_e ); free(vare_e); free(azel_e);
    free(svh_e); free(vsat_e); free(resp_e);
    return stat;
}
```
:::

### 10.1.5 valsol()：对定位结果进行卡方检验和GDOP检验 

**1. 参数列表**

```c
/* args */
const double *azel  I  方位角、高度角
const int *vsat     I  观测卫星在当前定位时是否有效 (1*n)
int n               I  观测值个数
const prcopt_t *opt I  处理选项
const double *v     I  定位方程的右端部分，伪距残差
int nv              I  观测值数
int nx              O  待估计参数数
/* return */
int                    status (1:ok,0:error)
```

**2. 执行流程**

$\begin{equation}
v_s = \frac{p_r^s - (\hat{\rho}_i^s + c\hat{d}t_r - cdT^s + I_r^s + T_r^s)}{\sigma_s} \tag{10.1}
\end{equation}$

$\begin{equation}
\mathbf{v} = (v_1, v_2, v_3, \ldots, v_m)^T \tag{10.2}
\end{equation}$

$\begin{equation}
\frac{v^T v}{m-n-1} < \chi^2_\alpha(m - n - 1) \tag{10.3}
\end{equation}$

$\begin{equation}
GDOP < GDOP_{thres} \tag{10.4}
\end{equation}$

> 其中 $n$ 是待估参数的数目， $m$ 是观测量的数目。$\chi_a^2(n)$ 是自由度 $n$ 和 $\alpha=0.001 (0.1\%)$ 的卡方分布。GDOP是几何精度因子（dilution of precision）。$GDOP_{thres}$ 可以在“Reject Threshold of GDOP”选项中进行相关配置。

以下为代码的执行流程：

- 先计算定位后伪距残余平方加权和 `vv`。
- 检查是否满足 `vv>chisqr[nv-nx-1]` ，满足条件则说明此时的定位解误差过大，返回 0；否则转到下一步。
- 复制 `azel` ，这里只复制那些对于定位结果有贡献的卫星的 `zael` 值，并且统计实现定位时所用卫星的数目。
- 调用 `dops` 函数，计算各种精度因子(DOP)，检验是否有 `0<GDOP<max` 。否，则说明该定位解的精度不符合要求，返回 0；是，则返回 1。

**3. 注意事项**

- 低成本接收机可能无法通过卡方校验，可禁用该部分内容（注释掉 `return 0` ）；
- 代码中的检验与 RTKLIB manual 中不一致（但与教材一致），文档中满足 $\frac{v^T v}{m-n-1} > \chi^2_\alpha(m - n - 1)$ 时就会不合格。与文档中相比，这里的写法将会放宽对于位置解的检验。

::: details 点击查看代码
```c
/* validate solution ---------------------------------------------------------*/
static int valsol(const double *azel, const int *vsat, int n,
                  const prcopt_t *opt, const double *v, int nv, int nx,
                  char *msg)
{
    double azels[MAXOBS*2],dop[4],vv;
    int i,ns;
    
    trace(3,"valsol  : n=%d nv=%d\n",n,nv);
    
    /* Chi-square validation of residuals */
    vv=dot(v,v,nv);
    // (E.6.33) 且观测值数大于待估计参数数  nv-nx-1:多余观测数
    // chisqr：卡方值表
    if (nv>nx&&vv>chisqr[nv-nx-1]) {
        sprintf(msg,"Warning: large chi-square error nv=%d vv=%.1f cs=%.1f",nv,vv,chisqr[nv-nx-1]);
        /* return 0; */ /* threshold too strict for all use cases, report error but continue on */
    }

    /* large GDOP check */
    for (i=ns=0;i<n;i++) {
        if (!vsat[i]) continue;
        azels[  ns*2]=azel[  i*2];
        azels[1+ns*2]=azel[1+i*2];
        ns++;
    }
    dops(ns,azels,opt->elmin,dop);
    if (dop[0]<=0.0||dop[0]>MAX_GDOP) {
        sprintf(msg,"gdop error nv=%d gdop=%.1f",nv,dop[0]);
        return 0;
    }
    return 1;
}
```
:::

### 10.1.6 lsq(): 最小二乘估计

**1. 参数列表**

```c
/* args */
double *A        I   设计矩阵的转置（可含权） (n x m)
double *y        I   观测残差项（可含权） (m x 1)
int    n,m       I   估计参数与观测量的维度 (n<=m)
double *x        O   估计参数 (n x 1)
double *Q        O   协方差阵 (n x n)
/* return */
int                  status (1:ok,0>:error)
```

**2. 执行过程**

- 首先计算右半部分 Ay=A*y 。
- 再计算左半部分括号里面的值 Q=A*A' 。
- 计算 Q 矩阵的逆 Q^-1，但仍存储在 Q 中，最后再右乘 Ay，得到 x 的值。

**3. 注意事项**

- **关于权重的运算**：对于加权最小二乘，可以直接在调用该函数之前直接将 A、y进行加权处理，之后在调用该函数，这样得到的就是加权最小二乘的解。
- **RTKLIB的列存储特性**：所有的矩阵都是列优先存储的，对于整个源代码来说，矩阵都是这样存储的。所以对于代码中出现的一维矩阵，基本都应该是列向量。在阅读数组下标时，记住这一点是非常重要的。
- **矩阵求逆的特性**：矩阵求逆并不简单，尤其是对于接近奇异的矩阵。但是由于这是个基本功能，并不打算继续深入下去。

::: details 点击查看代码
```c
/* least square estimation -----------------------------------------------------
* least square estimation by solving normal equation (x=(A*A')^-1*A*y)
* args   : double *A        I   transpose of (weighted) design matrix (n x m)
*          double *y        I   (weighted) measurements (m x 1)
*          int    n,m       I   number of parameters and measurements (n<=m)
*          double *x        O   estimated parameters (n x 1)
*          double *Q        O   estimated parameters covariance matrix (n x n)
* return : status (0:ok,0>:error)
* notes  : for weighted least square, replace A and y by A*w and w*y (w=W^(1/2))
*          matrix stored by column-major order (fortran convention)
*-----------------------------------------------------------------------------*/
extern int lsq(const double *A, const double *y, int n, int m, double *x,
               double *Q)
{
    double *Ay;
    int info;
    
    if (m<n) return -1;
    Ay=mat(n,1);
    matmul("NN",n,1,m,1.0,A,y,0.0,Ay); /* Ay=A*y */
    matmul("NT",n,n,m,1.0,A,A,0.0,Q);  /* Q=A*A' */
    if (!(info=matinv(Q,n))) matmul("NN",n,1,n,1.0,Q,Ay,0.0,x); /* x=Q^-1*Ay */
    free(Ay);
    return info;
}
```
:::

## 10.2 卫星相关函数

### 10.2.1 satsys()：根据卫星编号确定系统与PRN号

RTKLIB 为了方便矩阵的构建与运算，内部对卫星进行了顺序编号（satellite number）。

`satsys()` 传入卫星satellite number，返回卫星系统和 PRN 号，可以通过[笔者编写的转换工具](/algorithm/RTKLIB-Manual-CN/12-appendixH.html#h-1-prn-与-sat-编号对应关系)来直观感受这个过程。

**1. 参数列表**

```c
/* args */
int     sat      I   卫星编号 (1-MAXSAT)
int    *prn      IO  卫星 PRN 号或槽号 (NULL则无输出)
/* return */
int                  卫星系统 (SYS_GPS,SYS_GLO,...)
```

**2. 执行过程**

- 为处理意外情况（卫星号不在 1-MAXSAT之内），先令卫星系统为 SYS_NONE。
- 按照 RTKLIB 中定义相应宏的顺序来判断是否是 GPS、GLO、GAL系统等，判断标准是将卫星号减去前面的导航系统所拥有的卫星个数，来判断剩余卫星个数是否小于等于本系统的卫星个数。
- 确定所属的系统后，通过加上最小卫星编号的 PRN再减去 1，得到该卫星在该系统中的 PRN编号。

**3. 注意事项**

- 这里的卫星号是从 1开始排序的，这也是很多函数中与之有关的数组在调用时形式写为 `A[B.sat-1]`。

::: details 点击查看代码
```c
/* satellite number to satellite system ----------------------------------------
* convert satellite number to satellite system
* args   : int    sat       I   satellite number (1-MAXSAT)
*          int    *prn      IO  satellite prn/slot number (NULL: no output)
* return : satellite system (SYS_GPS,SYS_GLO,...)
*-----------------------------------------------------------------------------*/
extern int satsys(int sat, int *prn)
{
    int sys=SYS_NONE;
    if (sat<=0||MAXSAT<sat) sat=0;
    else if (sat<=NSATGPS) {
        sys=SYS_GPS; sat+=MINPRNGPS-1;
    }
    else if ((sat-=NSATGPS)<=NSATGLO) {
        sys=SYS_GLO; sat+=MINPRNGLO-1;
    }
    else if ((sat-=NSATGLO)<=NSATGAL) {
        sys=SYS_GAL; sat+=MINPRNGAL-1;
    }
    else if ((sat-=NSATGAL)<=NSATQZS) {
        sys=SYS_QZS; sat+=MINPRNQZS-1;
    }
    else if ((sat-=NSATQZS)<=NSATCMP) {
        sys=SYS_CMP; sat+=MINPRNCMP-1;
    }
    else if ((sat-=NSATCMP)<=NSATIRN) {
        sys=SYS_IRN; sat+=MINPRNIRN-1;
    }
    else if ((sat-=NSATIRN)<=NSATLEO) {
        sys=SYS_LEO; sat+=MINPRNLEO-1;
    }
    else if ((sat-=NSATLEO)<=NSATSBS) {
        sys=SYS_SBS; sat+=MINPRNSBS-1;
    }
    else sat=0;
    if (prn) *prn=sat;
    return sys;
}
```
:::

### 10.2.2 satexclude()：检测是否排除某颗卫星

**1. 参数列表**

```c
函数参数，3个
int       sat     I   卫星编号，从 1开始
int       svh     I   sv 健康标识 (0:ok)
prcopt_t  *opt    I   配置参数 (NULL: not used)
返回类型：
int               O   1:排除, 0:不排除
```

**2. 执行过程**

- 首先调用 `satsys` 函数得到该卫星所属的导航系统。
- 接着检验 `svh<0` 。是，则说明在 `ephpos` 函数中调用 `seleph` 为该卫星选择星历时，并无合适的星历可用，返回 1；否，则进入下一步。
- 如果处理选项不为空，则检测处理选项中对该卫星的排除标志的值(1:excluded,2:included)，之后再检测该卫星所属的导航系统与处理选项中预先设定的是否一致。否，返回 1；是，进入下一步。
- 如果此时 `svh>0`，说明此时卫星健康状况出现问题，此卫星不可用，返回 1。

**3. 问题思考**

- **satexclude的作用机制**：对于步骤3中检测，先验证状态排除标志，后验证导航系统，这样就可能出现排除标志符合要求而所属系统不符合要求的状况，而步骤3中做法会将上述状况设为 included。又或者，在步骤3中检测之后仍验证了 `svh>0`，那如果出现 `svh` 不合乎要求而排除标志符合要求的状况，步骤3中做法却会将上述状况设为 included。**该部分内容可以参考附录B.3**。

::: details 点击查看代码
```c
extern int satexclude(int sat, double var, int svh, const prcopt_t *opt)
{
   int sys=satsys(sat,NULL);
   
   // 通过svh判断卫星是否健康可用
   /* ephemeris unavailable */
   if (svh<0) return 1; 
   
   if (opt) {
       if (opt->exsats[sat-1]==1) return 1; /* excluded satellite */
       if (opt->exsats[sat-1]==2) return 0; /* included satellite */
       if (!(sys&opt->navsys)) return 1;    /* unselected sat sys */ // 比较该卫星与预先规定的导航系统是否一致
   }
   if (sys==SYS_QZS) svh&=0xFE; /* mask QZSS LEX health */
   if (svh) {
       trace(3,"unhealthy satellite: sat=%3d svh=%02X\n",sat,svh);
       return 1;
   }
   if (var>MAX_VAR_EPH) {
       trace(3,"invalid ura satellite: sat=%3d ura=%.2f\n",sat,sqrt(var));
       return 1;
   }
   return 0;
}
```
:::

### 10.2.3 testsnr()：检测信号是否可用

**1. 参数列表**

```c
/* args */
int    base      I   rover or base-station (0:rover,1:base station)
int    freq      I   frequency (0:L1,1:L2,2:L3,...)
double el        I   elevation angle (rad)
double snr       I   C/N0 (dBHz)
snrmask_t *mask  I   SNR mask
/* return */
int                  mask (1:masked,0:unmasked)
```

**2. 执行流程**

- 满足下列情况之一 `!mask->ena[base]||freq<0||freq>=NFREQ`，返回 0.
- 对 `el` 处理变换，根据后面 `i` 的值得到不同的阈值 `minsnr` ，而对 `1<=i<=8` 的情况，则使用线性插值计算出 `minsnr` 的值。

::: details 点击查看代码
```c
extern int testsnr(int base, int idx, double el, double snr,
                  const snrmask_t *mask)
{
    double minsnr,a;
    int i;
    
    if (!mask->ena[base]||idx<0||idx>=NFREQ) return 0;
    
    a=(el*R2D+5.0)/10.0;
    i=(int)floor(a); a-=i;
    if (i<1) {
       minsnr=mask->mask[idx][0];
    }
    else if (i>8) {
       minsnr=mask->mask[idx][8];
    }
    else {
       // 1<=i<=8 的情况，则使用线性插值计算出 minsnr 的值
       minsnr=(1.0-a)*mask->mask[idx][i-1]+a*mask->mask[idx][i];
    }
    
    return snr<minsnr;  // snr 小于 minsnr 时，返回1
}
```
:::

## 10.3 观测数据处理

### 10.3.1 rescode()：残差计算、设计矩阵构建

计算当前迭代的伪距残差 `v`、设计矩阵 `H`、伪距残差的方差 `var`、所有观测卫星的方位角和仰角 `azel`，定位时有效性 `vsa`t、定位后伪距残差 `resp`、参与定位的卫星个数 `ns` 和方程个数 `nv` 。

> `dion`,`dtrp`,`vmeas`,`vion`,`vtrp`四个局部变量没有初始化， 运行时会报错，可赋初值0.0

**1. 参数列表**

::: details 点击查看代码
```c
/* args */
int      iter      I   迭代次数，在estpos()里迭代调用，第i次迭代就传i
obsd_t   *obs      I   观测量数据
int      n         I   观测量数据的数量
double   *rs       I   卫星位置和速度，长度为6*n，{x,y,z,vx,vy,vz}(ecef)(m,m/s)
double   *dts      I   卫星钟差，长度为2*n， {bias,drift} (s|s/s)
double   *vare     I   卫星位置和钟差的协方差 (m^2)
int      *svh      I   卫星健康标志 (-1:correction not available)
nav_t    *nav      I   导航数据
double   *x        I   本次迭代开始之前的定位值,7*1,前3个是本次迭代开始之前的定位值，第4个是钟差，后三个分别是gps系统与glonass、galileo、bds系统的钟差。
prcopt_t *opt      I   处理过程选项
double   *v        O   定位方程的右端部分，伪距残差
double   *H        O   定位方程中的几何矩阵
double   *var      O   参与定位的伪距残差的方差
double   *azel     O   对于当前定位值，所有观测卫星的 {方位角、高度角} (2*n)
int      *vsat     O   所有观测卫星在当前定位时是否有效 (1*n)
double   *resp     O   所有观测卫星的伪距残差，(P-(r+c*dtr-c*dts+I+T)) (1*n)
int      *ns       O   参与定位的卫星的个数
/* return */
int                    nv (定位方程组的方程个数)
```
:::

**2. 执行流程**

* 将上一历元的定位信息赋值给`rr`和`dtr`数组。
* 调用`ecef2pos()`将将接收机位置`rr`由 ECEF-XYZ 转换为大地坐标系LLH`pos`
* 遍历当前历元所有`obs[]`，即遍历每颗卫星：
  * 将`vsat[]`、`azel[]`和`resp[]`数组置 0，因为在前后两次定位结果中，每颗卫星的上述信息都会发生变化。`time`赋值obs的时间，`sat`赋值obs的卫星。
  * 检测当前观测卫星是否和下一个相邻数据重复；重复则不处理这一条，continue去处理下一条。
  * 调用`satexclude()`函数判断卫星是否需要排除，如果排除则continue去处理下一个卫星。
  * 调用`geodist()`函数，计算卫星和当前接收机位置之间的几何距离`r`和接收机到卫星的方向向量`e`。 
  * 调用`satazel()`函数，计算在接收机位置处的站心坐标系中卫星的方位角和仰角；若仰角低于截断值`opt->elmin`，continue不处理此数据。
  * 调用`snrmask()`，根据接收机高度角和信号频率来检测该信号是否可用。
  * 调用` ionocorr()` 函数，计算电离层延时`I`,所得的电离层延时是建立在 L1 信号上的，当使用其它频率信号时，依据所用信号频组中第一个频率的波长与 L1 波长的比率，对上一步得到的电离层延时进行修正。 
  * 调用`tropcorr()`函数,计算对流层延时`T`。
  * 调用`prange()`函数，计算经过DCB校正后的伪距值`p`。
  * 计算伪距残差`v[nv]`，即经过钟差，对流层，电离层改正后的伪距。
  * 构建设计矩阵 `H`

    单点定位的测量方程及其偏导数矩阵构成如下：

    $\begin{equation}
    \mathbf{h(x)} = 
    \begin{pmatrix} 
    \rho_r^1 + cdt_r - cdT^1 + I_r^1 + T_r^1 \\ 
    \rho_r^2 + cdt_r - cdT^2 + I_r^2 + T_r^2 \\ 
    \rho_r^3 + cdt_r - cdT^3 + I_r^3 + T_r^3 \\ 
    \vdots \\
    \rho_r^m + cdt_r - cdT^m + I_r^m + T_r^m 
    \end{pmatrix} \mathbf{H} = 
    \begin{pmatrix} 
    -\mathbf{e}_r^{1^T} & 1 \\ 
    -\mathbf{e}_r^{2^T} & 1 \\ 
    -\mathbf{e}_r^{3^T} & 1 \\ 
    \vdots & \vdots \\ 
    -\mathbf{e}_r^{m^T} & 1
    \end{pmatrix} \tag{10.5}
    \end{equation}$

    公式摘录自 [RTKLIB Manual](/algorithm/RTKLIB-Manual-CN/09-appendixE-E.6.html)，其中几何距离 $\rho_r^s$ 和视距（LOS）向量 $\mathbf{e}_r^s$ 由 E.3 (3.4) 和 E.3 (3.5) 给出，结合卫星和接收机的位置。卫星位置 $\mathbf{r}^s$ 和钟差 $dT^s$ 则根据配置选项“Satellite Ephemeris/Clock”从 E.4 中描述的GNSS卫星星历和时钟推导而来。

  * 处理不同系统（GPS、GLO、GAL、CMP）之间的时间偏差，修改矩阵`H `。
  * 调用`varerr()`函数，计算此时的导航系统误差
* 为了防止不满秩的情况，把矩阵`H`补满秩了，`H[j+nv*NX]=j==i+3?1.0:0.0; `

**3. 注意事项**

- 返回值 `v` 和 `resp` 的主要区别在于长度不一致， `v` 是需要参与定位方程组的解算的，维度为 `nv*1` ；而 `resp` 仅表示所有观测卫星（包含不健康的卫星）的伪距残余，维度为 `n*1`，对于没有参与定位的卫星，该值为 0，而 nv 小于 n。
- 源码中 `dtr` 的单位是 m。
- 调用`varerr()`函数时候，其中的 `URE` 值包括：
  - a. 卫星星历和钟差的误差
  - b. 大气延时误差
  - c. 伪距测量的码偏移误差
  - d. 导航系统的误差

::: details 点击查看代码
```c
static int rescode(int iter, const obsd_t *obs, int n, const double *rs,
                 const double *dts, const double *vare, const int *svh,
                 const nav_t *nav, const double *x, const prcopt_t *opt,
                 double *v, double *H, double *var, double *azel, int *vsat,
                 double *resp, int *ns)
{
  gtime_t time;
  double r, freq, dion = 0.0, dtrp = 0.0, vmeas, vion = 0.0, vtrp = 0.0, rr[3], pos[3], dtr, e[3], P;
  int i,j,nv=0,sat,sys,mask[NX-3]={0};
  
  trace(3,"resprng : n=%d\n",n);

  // 将之前得到的定位解信息赋值给 rr 和 dtr 数组，以进行关于当前解的伪距残差的相关计算
  for (i=0;i<3;i++) rr[i]=x[i];
  dtr=x[3];               
  
  ecef2pos(rr,pos);   // rr{x,y,z}->pos{lat,lon,h}     
  
  // 遍历当前历元所有obs[]         
  for (i=*ns=0;i<n&&i<MAXOBS;i++) {
      // 将vsat、azel和resp数组置 0，因为在前后两次定位结果中，每颗卫星的上述信息都会发生变化。
      vsat[i]=0; azel[i*2]=azel[1+i*2]=resp[i]=0.0;   
      time=obs[i].time;       // time赋值obs的时间
      sat=obs[i].sat;         // sat赋值obs的卫星
      
      // 调用satsys()函数，验证卫星编号是否合理及其所属的导航系统
      if (!(sys=satsys(sat,NULL))) continue;  
     
      // 检测当前观测卫星是否和下一个相邻数据重复；重复则不处理这一条，去处理下一条
      /* reject duplicated observation data */
      if (i<n-1&&i<MAXOBS-1&&sat==obs[i+1].sat) {
          trace(2,"duplicated obs data %s sat=%d\n",time_str(time,3),sat);
          i++;
          continue;
      }

      // 处理选项中事先指定定位时排除哪些导航系统或卫星,调用 satexclude 函数完成
      /* excluded satellite? */
      if (satexclude(sat,vare[i],svh[i],opt)) continue;

      // 调用 geodist 函数，计算卫星和当前接收机位置之间的几何距离 r和接收机到卫星方向的观测矢量。
      // 然后检验几何距离是否 >0。此函数中会进行地球自转影响的校正（Sagnac效应）
      /* geometric distance */
      if ((r=geodist(rs+i*6,rr,e))<=0.0) continue;
     
      if (iter>0) {
          // 调用 satazel 函数，计算在接收机位置处的站心坐标系中卫星的方位角和仰角；若仰角低于截断值，不处理此数据。
          /* test elevation mask */
          if (satazel(pos,e,azel+i*2)<opt->elmin) continue;
         
          // 调用snrmask()->testsnr()，根据接收机高度角和信号频率来检测该信号是否可用
          /* test SNR mask */
          if (!snrmask(obs+i,azel+i*2,opt)) continue;

          // 调用 ionocorr 函数，计算电离层延时I,所得的电离层延时是建立在 L1 信号上的，
          // 当使用其它频率信号时，依据所用信号频组中第一个频率的波长与 L1 波长的关系，对上一步得到的电离层延时进行修正。
          /* ionospheric correction */
          if (!ionocorr(time,nav,sat,pos,azel+i*2,opt->ionoopt,&dion,&vion)) {
              continue;
          }
          if ((freq=sat2freq(sat,obs[i].code[0],nav))==0.0) continue;
          dion*=SQR(FREQ1/freq);
          vion*=SQR(FREQ1/freq);

          // 调用 tropcorr 函数，计算对流层延时T
          /* tropospheric correction */
          if (!tropcorr(time,nav,pos,azel+i*2,opt->tropopt,&dtrp,&vtrp)) {
              continue;
          }
      }
      // 调用 prange 函数，得到经过DCB校正后的伪距值 ρ
      /* psendorange with code bias correction */
      if ((P=prange(obs+i,nav,opt,&vmeas))==0.0) continue;
      
      // 伪距残差 //(P-(r+c*dtr-c*dts+I+T))   (E.6.31)
      /* pseudorange residual */
      v[nv]=P-(r+dtr-CLIGHT*dts[i*2]+dion+dtrp);  
     
      // 构建设计矩阵H单位向量的反，前 3 行为中计算得到的视线向，第 4 行为 1，其它行为 0
      /* design matrix */
      for (j=0;j<NX;j++) {
          H[j+nv*NX]=j<3?-e[j]:(j==3?1.0:0.0);
      }
     
      // 处理不同系统（GPS、GLO、GAL、CMP）之间的时间偏差，修改矩阵 H
      /* time system offset and receiver bias correction */
      if      (sys==SYS_GLO) {v[nv]-=x[4]; H[4+nv*NX]=1.0; mask[1]=1;}
      else if (sys==SYS_GAL) {v[nv]-=x[5]; H[5+nv*NX]=1.0; mask[2]=1;}
      else if (sys==SYS_CMP) {v[nv]-=x[6]; H[6+nv*NX]=1.0; mask[3]=1;}
      else if (sys==SYS_IRN) {v[nv]-=x[7]; H[7+nv*NX]=1.0; mask[4]=1;}
#if 0 /* enable QZS-GPS time offset estimation */
      else if (sys==SYS_QZS) {v[nv]-=x[8]; H[8+nv*NX]=1.0; mask[5]=1;}
#endif
      else mask[0]=1;
     
      vsat[i]=1; resp[i]=v[nv]; (*ns)++;
      // 调用 varerr 函数，计算此时的导航系统误差，然后累加计算用户测距误差(URE)。
      /* variance of pseudorange error */
      var[nv++]=varerr(opt,azel[1+i*2],sys)+vare[i]+vmeas+vion+vtrp;
     
      trace(4,"sat=%2d azel=%5.1f %4.1f res=%7.3f sig=%5.3f\n",obs[i].sat,
           azel[i*2]*R2D,azel[1+i*2]*R2D,resp[i],sqrt(var[nv-1]));
  }

  // 防止秩亏：状态中包含不同系统的钟差信息，如果H矩阵不进处理，那么其对应列会出现全0的情况
  // 注意：H实际是最小二乘设计矩阵的转置
  /* constraint to avoid rank-deficient */
  for (i=0;i<NX-3;i++) {
      if (mask[i]) continue;
      v[nv]=0.0;
      for (j=0;j<NX;j++) H[j+nv*NX]=j==i+3?1.0:0.0;  
      var[nv++]=0.01;
  }

  // 返回值 v和 resp的主要区别在于长度不一致， v是需要参与定位方程组的解算的，维度为 nv*1；
  // 而resp仅表示所有观测卫星的伪距残余，维度为 n*1，对于没有参与定位的卫星，该值为 0
  return nv;
}
```
:::

### 10.3.2 resdop()：定速方程残差计算、设计矩阵构建

计算定速方程组左边的几何矩阵和右端的速度残余，返回定速时所使用的卫星数目。

**1. 参数列表**：

```c
/* args */
obsd_t   *obs      I   观测量数据
int      n         I   观测量数据的数量
double   *rs       I   卫星位置和速度，长度为6*n，{x,y,z,vx,vy,vz}(ecef)(m,m/s)
double   *dts      I   卫星钟差，长度为2*n， {bias,drift} (s|s/s)
nav_t    *nav      I   导航数据
double   *rr       I   接收机位置和速度，长度为6，{x,y,z,vx,vy,vz}(ecef)(m,m/s)
double   *x        I   本次迭代开始之前的定速值，长度为4，{vx,vy,vz,drift}
double   *azel     IO  方位角和俯仰角 (rad)
int      *vsat     I   卫星在定速时是否有效
double   *v        O   定速方程的右端部分，速度残差
double   *H        O   定速方程中的几何矩阵
/* return */
int                    nv (定速时所使用的卫星数目)
```

**2. 执行流程**

* 调用`ecef2pos()`函数，将接收机位置由 ECEF-XYZ 转换为大地坐标系LLH，调用`xyz2enu()`函数，计算到ENU坐标转换矩阵`E`。 
* 遍历当前历元每一个观测值（卫星），即遍历所有卫星：
  * 去除在定速时不可用的卫星。 
  * 计算当前接收机位置下 ENU中的视向量`e`，然后用刚刚计算出的转换矩阵`E`转换得到 ECEF 中视向量的值。 
  * 计算ECEF中卫星相对于接收机的速度，卫星速度`rs[j+3+i*6]`-传入的定速初值`x[j] `。
  * 计算考虑了地球自转的用户和卫星之间的几何距离变化率`rate `。
  * 计算`rate`的标准差、残差。
  * 构建设计矩阵 `H`，将观测方程数 `nv` 加1

    RTKLIB总是使用 $L_1$ 频段的多普勒频率观测值。这些测量方程及其偏导数矩阵构成如下：

    $\begin{equation}
    \mathbf{h(x)} = \begin{pmatrix}
    r_r^1 + cd\dot{t}_r - cd\dot{T}^1 \\
    r_r^2 + cd\dot{t}_r - cd\dot{T}^2 \\
    r_r^3 + cd\dot{t}_r - cd\dot{T}^3 \\
    \vdots \\
    r_r^m + cd\dot{t}_r - cd\dot{T}^m
    \end{pmatrix} \quad
    \mathbf{H} = \begin{pmatrix}
    -\mathbf{e}_r^{1^T} & 1 \\
    -\mathbf{e}_r^{2^T} & 1 \\
    -\mathbf{e}_r^{3^T} & 1 \\
    \vdots & \vdots \\
    -\mathbf{e}_r^{m^T} & 1
    \end{pmatrix} \tag{10.6}
    \end{equation}$

    这些方程中卫星相对于接收机的速度 $\mathbf{r}_r^s$ 从以下公式推导：

    $\begin{equation}
    \mathbf{r}_r^s = \mathbf{e}_r^{s^T} (\mathbf{v}^s(t^s) - \mathbf{v}_r) + \frac{\omega_e}{c} (v_y^s x_r + y^s v_{x,r} - v_x^s y_r - x^sv_{y,r}) \tag{10.7}
    \end{equation}$

    其中 $\mathbf{v}^s = (v_x^s, v_y^s, v_z^s)^T$ ， $\mathbf{v}_r = (v_{x,r}, v_{y,r}, v_{z,r})^T$ 。

* 遍历完每一个观测值，返回定数方程数`nv`。

**3. 注意事项**

- **亏秩问题**：这里与定位不同，构建几何矩阵时，就只有4个未知数，而定位时是有 NX 个。并且没有像定位那样为了防止亏秩而进行约束处理。
- **几何矩阵**：多普勒定速方程中几何矩阵 G 与定位方程中的一样，前三行都是 ECEF 坐标系中由接收机指向卫星的单位观测矢量的反向。而由于转换矩阵 S 本身是一个正交单位矩阵，所以这里在计算 ECEF 中的视向量时，对 E 进行了转置处理。
- 这里构建的左端几何矩阵 H，也与定位方程中的一样，是大部分资料上的几何矩阵的转置。

:::details 点击查看代码
```c
static int resdop(const obsd_t *obs, int n, const double *rs, const double *dts,
                 const nav_t *nav, const double *rr, const double *x,
                 const double *azel, const int *vsat, double err, double *v,
                 double *H)
{
   double freq,rate,pos[3],E[9],a[3],e[3],vs[3],cosel,sig;
   int i,j,nv=0;
   
   trace(3,"resdop  : n=%d\n",n);
   
   // 调用 ecef2pos() 函数，将接收机位置由 ECEF 转换为大地坐标系。
   // 调用 xyz2enu() 函数，计算到ENU坐标转换矩阵 E。
   ecef2pos(rr,pos); xyz2enu(pos,E);  
   
   for (i=0;i<n&&i<MAXOBS;i++) {
       
       freq=sat2freq(obs[i].sat,obs[i].code[0],nav);
       
       // 去除在定速时不可用的卫星。
       if (obs[i].D[0]==0.0||freq==0.0||!vsat[i]||norm(rs+3+i*6,3)<=0.0) {
           continue;
       }
       // 计算当前接收机位置下 ENU 中的视向量 e，然后转换得到 ECEF 中视向量的值。
       /* LOS (line-of-sight) vector in ECEF */    
       cosel=cos(azel[1+i*2]);
       a[0]=sin(azel[i*2])*cosel;
       a[1]=cos(azel[i*2])*cosel;
       a[2]=sin(azel[1+i*2]);              
       matmul("TN",3,1,3,1.0,E,a,0.0,e);   
       
       // 计算 ECEF 中卫星相对于接收机的速度，卫星速度rs[j+3+i*6]-传入的定速初值x[j]
       /* satellite velocity relative to receiver in ECEF */
       for (j=0;j<3;j++) {         
           vs[j]=rs[j+3+i*6]-x[j]; 
       }                           

       // 计算考虑了地球自转的用户和卫星之间的几何距离变化率
       /* range rate with earth rotation correction */
       rate=dot(vs,e,3)+OMGE/CLIGHT*(rs[4+i*6]*rr[0]+rs[1+i*6]*x[0]-
                 rs[3+i*6]*rr[1]-rs[  i*6]*x[1]);	//(F.6.29)

       /* Std of range rate error (m/s) */     
       sig=(err<=0.0)?1.0:err*CLIGHT/freq;     
       
       /* range rate residual (m/s) */
       v[nv]=(-obs[i].D[0]*CLIGHT/freq-(rate+x[3]-CLIGHT*dts[1+i*2]))/sig;
       
       /* design matrix */ // 构建左端项的几何矩阵
       for (j=0;j<4;j++) {
           H[j+nv*4]=((j<3)?-e[j]:1.0)/sig; // (E.6.28)
       }
       nv++; // 将观测方程数增 1
   }
   return nv;
}
```
:::

### 10.3.3 varerr()：计算导航系统伪距测量值的误差

`varerr()`一定程度上代表了误差的水平，该函数是衡量载波相位误差的函数，通过fact因子（如300）可以放缩到伪距误差的水平。对于权重矩阵 $\mathbf{W}$ ，RTKLIB使用以下公式：

$\begin{equation}
\mathbf{W} = diag(\sigma_1^{-2}, \sigma_2^{-2}, \ldots, \sigma_m^{-2}) \tag{10.7}
\end{equation}$

$\begin{equation}
\sigma^2 = {F^s}^2 R_r^2 \left(a_\sigma^2 + b_\sigma^2 / \sin El_r^s \right) + \sigma_{eph}^2 + \sigma_{ion}^2 + \sigma_{trop}^2 + \sigma_{bias}^2 \tag{10.8}
\end{equation}$

其中：

$F^s$：卫星系统误差因子<br>
（1: GPS, Galileo, QZSS和Beidou, 1.5: GLONASS, 3.0: SBAS）<br>
$R_r$：码/载波相位误差比率，伪距定位和载波相位定位使用的是同样的加权公式，这里以载波相位为基础，然后通过系数转换到了伪距上<br>
$a_\sigma, b_\sigma$：载波相位误差因子 $a$ 和 $b$（m）<br>
$\sigma_{eph}$：星历和钟差标准差（m）<br>
$\sigma_{ion}$：电离层校正模型误差标准差（m）<br>
$\sigma_{trop}$：对流层校正模型误差标准差（m）<br>
$\sigma_{bias}$：码偏差误差标准差（m）

**1. 参数列表**

```c
/* args */
prcopt_t  *opt    I   processing options
double     el     I   elevation angle (rad)
int        sys    I   所属的导航系统
/* return */
double                varr (导航系统伪距测量值的误差)
```

**2. 执行流程**

- 确定 sys系统的误差因子。
- 计算由导航系统本身所带来的误差的方差。
- 如果 ionoopt==IONOOPT_IFLC时，IFLC模型的方差也会添加到最终计算得到的方差中。

**3. 问题思考**

- IFLC 模型的方差为什么使用 `varr*=SQR(3.0)` 计算？该部分参考附录B.4。

::: details 点击查看代码
```c
/* 这里的代码使用的是demo5版本 */
static double varerr(const prcopt_t *opt, const ssat_t *ssat, const obsd_t *obs, double el, int sys)
{
    double fact=1.0,varr,snr_rover;

    switch (sys) {
        case SYS_GPS: fact *= EFACT_GPS; break; // fact=1.0
        case SYS_GLO: fact *= EFACT_GLO; break; // fact=1.5
        case SYS_SBS: fact *= EFACT_SBS; break; // fact=3.0
        case SYS_CMP: fact *= EFACT_CMP; break; // fact=1.0
        case SYS_QZS: fact *= EFACT_QZS; break; // fact=1.0
        case SYS_IRN: fact *= EFACT_IRN; break; // fact=1.5
        default:      fact *= EFACT_GPS; break; // fact=1.0
    }

    // 高度角过小，设为5°
    if (el<MIN_EL) el=MIN_EL;

    /* var = R^2*(a^2 + (b^2/sin(el) + c^2*(10^(0.1*(snr_max-snr_rover)))) + (d*rcv_std)^2) */
    varr=SQR(opt->err[1])+SQR(opt->err[2])/sin(el);
    if (opt->err[6]>0.0) {  /* if snr term not zero */
        snr_rover=(ssat)?SNR_UNIT*ssat->snr_rover[0]:opt->err[5];
        varr+=SQR(opt->err[6])*pow(10,0.1*MAX(opt->err[5]-snr_rover,0));
    }
    varr*=SQR(opt->eratio[0]);
    if (opt->err[7]>0.0) {
        varr+=SQR(opt->err[7]*0.01*(1<<(obs->Pstd[0]+5)));  /* 0.01*2^(n+5) m */
    }

    /* iono-free */ // 无电离层组合，方差*3*3
    if (opt->ionoopt==IONOOPT_IFLC) varr*=SQR(3.0);

    return SQR(fact)*varr;
}
```
:::

### 10.3.4 prange()：计算经过DCB校正后的伪距值`p`

差分码偏差（DCB）是 GNSS 伪距观测中由不同测距码信号（如 C1、P1、P2）在卫星和接收机硬件通道中产生的时延差异。分为：

- **频内偏差**：同一频率不同码的时延差，如 P1-C1。
- **频间偏差**：不同频率码的时延差，如 P1-P2。

DCB 由信号生成至发射的硬件时延差异引起，校正 DCB 可提高 GNSS 定位精度。

**1. 参数列表**

```c
/* args */
obsd_t    *obs      I   观测数据
nav_t     *nav      I   导航数据
double    *azel     I   对于当前定位值，每一颗观测卫星的 {方位角、高度角}
int        iter     I   迭代次数
prcopt_t  *opt      I   配置参数
double    *vare     O   伪距测量的码偏移误差
/* return */
double                  P1 (最终参与定位解算的伪距值)
```

**2. 执行流程**

::: details 点击查看更多
- 初始化和基本检查
  - 获取卫星编号和系统类型（如 GPS、GLONASS）。
  - 获取 L1/E1/B1 伪距（P1）和第二频率伪距（P2，由 seliflc 选择，如 L2、L5）。
  - 初始化方差 *var = 0.0。
  - 检查观测值有效性：若 P1 为 0 或 IFLC 模式下 P2 为 0，返回 0.0（无效观测）。
- DCB 校正
  - L1/E1/B1 校正：通过 code2bias_ix 获取 L1 频率测距码的 DCB 索引（bias_ix），若非参考码（bias_ix > 0），校正 P1。
  - L2/L5 校正：对第二频率伪距（P2）进行类似校正，但 Galileo 的 L2（f2 == 1）无 DCB 数据，跳过校正。
  - DCB 数据存储在 nav->cbias，按系统、频率和码类型索引。
- 双频无电离层组合（IFLC）
  - GPS/QZSS（L1-L2 或 L1-L5）
    ```c
    // 使用 L1/L2（FREQL1/FREQL2）或 L1/L5（FREQL1/FREQL5）频率比的平方
    gamma = f2 == 1 ? SQR(FREQL1/FREQL2) : SQR(FREQL1/FREQL5);
    return (P2 - gamma * P1) / (1.0 - gamma);
    ```
  - GLONASS（G1-G2 或 G1-G3）
    ```c
    // 使用 GLONASS 频率（如 G1/G2 或 G1/G3）
    gamma = f2 == 1 ? SQR(FREQ1_GLO/FREQ2_GLO) : SQR(FREQ1_GLO/FREQ3_GLO);
    return (P2 - gamma * P1) / (1.0 - gamma);
    ```
  - Galileo（E1-E5b 或 E1-E5a）
    ```c
    // 使用 E1/E5b 或 E1/E5a 频率比。
    // 对 F/NAV 星历（getseleph(SYS_GAL)），校正 E5a/E5b 的广播群延迟（BGD）。
    gamma = f2 == 1 ? SQR(FREQL1/FREQE5b) : SQR(FREQL1/FREQL5);
    if (f2 == 1 && getseleph(SYS_GAL)) {
        P2 -= gettgd(sat, nav, 0) - gettgd(sat, nav, 1); /* BGD_E5aE5b */
    }
    return (P2 - gamma * P1) / (1.0 - gamma);
    ```
  - BeiDou（B1-B2）
    ```c
    // 支持 B1I/B1Cp/B1Cd 码，校正 TGD（b1, b2）
    // 公式包含 TGD 校正：$P_{IF} = \frac{(P_2 - b_2) - \gamma (P_1 - b_1)}{1 - \gamma}$
    gamma = SQR(((obs->code[0] == CODE_L2I) ? FREQ1_CMP : FREQL1) / FREQ2_CMP);
    if (obs->code[0] == CODE_L2I) b1 = gettgd(sat, nav, 0); /* TGD_B1I */
    else if (obs->code[0] == CODE_L1P) b1 = gettgd(sat, nav, 2); /* TGD_B1Cp */
    else b1 = gettgd(sat, nav, 2) + gettgd(sat, nav, 4); /* TGD_B1Cp+ISC_B1Cd */
    b2 = gettgd(sat, nav, 1); /* TGD_B2I/B2bI (m) */
    return ((P2 - gamma * P1) - (b2 - gamma * b1)) / (1.0 - gamma);
    ```
  - IRNSS（L5-S）
- 单频伪距校正
  当非 IFLC 模式（单频），直接使用 L1/E1/B1 伪距，并校正 TGD：
  - GPS/QZSS：校正 L1 的 TGD（b1）。
  - GLONASS：校正 G1 的 TGD，考虑频率比 $\gamma$。
  - Galileo：校正 E1 的 BGD（E1-E5a 或 E1-E5b）。
  - BeiDou：根据码类型（B1I/B1Cp/B1Cd）校正 TGD。
  - IRNSS：校正 L5 的 TGD，考虑频率比。
  - 方差设为 SQR(ERR_CBIAS)，表示单频伪距的码偏差误差。
:::

**3. 注意事项**

- **prange 的直接作用**： 主要是是计算 GNSS 伪距观测值，经过差分码偏差（DCB）校正、广播群延迟（TGD/BGD）校正，并根据用户选项（`opt->ionoopt`）选择 无电离层线性组合（IFLC） 或 单频伪距 的处理方式。该函数并不直接针对 C/A 码（如 C1）进行特定修正，而是对所有测距码（如 C1、P1、P2 等）应用通用的 DCB 和 TGD 校正，以消除硬件时延和电离层效应。

:::details 点击查看代码
```c
/* 计算伪距（无电离层组合或单频），包含DCB和TGD/BGD校正 */
static double prange(const obsd_t *obs, const nav_t *nav, const prcopt_t *opt,
                     double *var)
{
    double P1, P2, gamma, b1, b2; /* P1, P2: 伪距; gamma: 频率比平方; b1, b2: TGD/BGD */
    int sat, sys, f2, bias_ix;    /* sat: 卫星编号; sys: 系统类型; f2: 第二频率索引; bias_ix: DCB索引 */

    /* 初始化变量 */
    sat = obs->sat;                           /* 获取卫星编号 */
    sys = satsys(sat, NULL);                  /* 获取卫星系统（如GPS、GLONASS） */
    P1 = obs->P[0];                           /* 获取L1/E1/B1伪距 */
    f2 = seliflc(opt->nf, satsys(obs->sat, NULL)); /* 选择第二频率（如L2、L5） */
    P2 = obs->P[f2];                          /* 获取L2/L5/E5等伪距 */
    *var = 0.0;                               /* 初始化方差 */

    /* 检查观测值有效性：P1为0或IFLC模式下P2为0，返回0.0 */
    if (P1 == 0.0 || (opt->ionoopt == IONOOPT_IFLC && P2 == 0.0)) return 0.0;

    /* 校正L1/E1/B1的DCB（频内偏差，如P1-C1） */
    bias_ix = code2bias_ix(sys, obs->code[0]); /* 获取L1测距码的DCB索引 */
    if (bias_ix > 0) {                         /* 非参考码（bias_ix=0为参考码） */
        P1 += nav->cbias[sat-1][0][bias_ix-1]; /* 应用L1 DCB校正 */
    }

    /* 校正L2/L5/E5的DCB（频内偏差，如P2-C2） */
    if (sys == SYS_GAL && f2 == 1) {
        /* Galileo L2无DCB数据，跳过校正 */
    } else {
        bias_ix = code2bias_ix(sys, obs->code[f2]); /* 获取第二频率测距码的DCB索引 */
        if (bias_ix > 0) {                          /* 非参考码 */
            P2 += nav->cbias[sat-1][1][bias_ix-1];  /* 应用L2/L5 DCB校正 */
        }
    }

    /* 双频无电离层组合（IFLC）处理 */
    if (opt->ionoopt == IONOOPT_IFLC) {
        if (sys == SYS_GPS || sys == SYS_QZS) { /* GPS/QZSS: L1-L2或L1-L5 */
            gamma = f2 == 1 ? SQR(FREQL1/FREQL2) : SQR(FREQL1/FREQL5); /* 计算频率比平方 */
            *var = SQR(3.0) * SQR(ERR_CBIAS); /* 近似方差放大（噪声放大因子≈3.0） */
            return (P2 - gamma * P1) / (1.0 - gamma); /* IFLC公式 */
        } else if (sys == SYS_GLO) { /* GLONASS: G1-G2或G1-G3 */
            gamma = f2 == 1 ? SQR(FREQ1_GLO/FREQ2_GLO) : SQR(FREQ1_GLO/FREQ3_GLO);
            *var = SQR(3.0) * SQR(ERR_CBIAS); /* 近似方差放大 */
            return (P2 - gamma * P1) / (1.0 - gamma);
        } else if (sys == SYS_GAL) { /* Galileo: E1-E5b或E1-E5a */
            gamma = f2 == 1 ? SQR(FREQL1/FREQE5b) : SQR(FREQL1/FREQL5);
            if (f2 == 1 && getseleph(SYS_GAL)) { /* F/NAV星历，校正BGD */
                P2 -= gettgd(sat, nav, 0) - gettgd(sat, nav, 1); /* BGD_E5aE5b */
            }
            *var = SQR(3.0) * SQR(ERR_CBIAS); /* 近似方差放大 */
            return (P2 - gamma * P1) / (1.0 - gamma);
        } else if (sys == SYS_CMP) { /* BeiDou: B1-B2 */
            gamma = SQR(((obs->code[0] == CODE_L2I) ? FREQ1_CMP : FREQL1) / FREQ2_CMP);
            if (obs->code[0] == CODE_L2I) b1 = gettgd(sat, nav, 0); /* TGD_B1I */
            else if (obs->code[0] == CODE_L1P) b1 = gettgd(sat, nav, 2); /* TGD_B1Cp */
            else b1 = gettgd(sat, nav, 2) + gettgd(sat, nav, 4); /* TGD_B1Cp+ISC_B1Cd */
            b2 = gettgd(sat, nav, 1); /* TGD_B2I/B2bI */
            *var = SQR(3.0) * SQR(ERR_CBIAS); /* 近似方差放大 */
            return ((P2 - gamma * P1) - (b2 - gamma * b1)) / (1.0 - gamma); /* IFLC含TGD校正 */
        } else if (sys == SYS_IRN) { /* IRNSS: L5-S */
            gamma = SQR(FREQL5/FREQs);
            *var = SQR(3.0) * SQR(ERR_CBIAS); /* 近似方差放大 */
            return (P2 - gamma * P1) / (1.0 - gamma);
        }
    } else { /* 单频（L1/E1/B1）处理 */
        *var = SQR(ERR_CBIAS); /* 单频方差，基于码偏差误差 */
        if (sys == SYS_GPS || sys == SYS_QZS) { /* GPS/QZSS: L1 */
            b1 = gettgd(sat, nav, 0); /* TGD */
            return P1 - b1;
        } else if (sys == SYS_GLO) { /* GLONASS: G1 */
            gamma = SQR(FREQ1_GLO/FREQ2_GLO);
            b1 = gettgd(sat, nav, 0); /* -dtaun */
            return P1 - b1 / (gamma - 1.0);
        } else if (sys == SYS_GAL) { /* Galileo: E1 */
            if (getseleph(SYS_GAL)) b1 = gettgd(sat, nav, 0); /* BGD_E1E5a */
            else b1 = gettgd(sat, nav, 1); /* BGD_E1E5b */
            return P1 - b1;
        } else if (sys == SYS_CMP) { /* BeiDou: B1I/B1Cp/B1Cd */
            if (obs->code[0] == CODE_L2I) b1 = gettgd(sat, nav, 0); /* TGD_B1I */
            else if (obs->code[0] == CODE_L1P) b1 = gettgd(sat, nav, 2); /* TGD_B1Cp */
            else b1 = gettgd(sat, nav, 2) + gettgd(sat, nav, 4); /* TGD_B1Cp+ISC_B1Cd */
            return P1 - b1;
        } else if (sys == SYS_IRN) { /* IRNSS: L5 */
            gamma = SQR(FREQs/FREQL5);
            b1 = gettgd(sat, nav, 0); /* TGD */
            return P1 - gamma * b1;
        }
    }
    return P1; /* 默认返回L1伪距 */
}
```
:::

## 10.4 坐标与几何距离

### 10.4.1 geodist()：计算站心几何距离

**1. 基本原理**

地球自转引起的误差源自于信号传输过程中：GNSS 卫星信号**发射时刻**和接收机**接收到信号的时刻**之间**地球自转对 GNSS 观测值产生的影响**。

相当于地球自转使得卫星空间位置在信号播发、接收过程中接收机在地固系坐标轴上相对于 z 轴发生了一定角度的旋转，使得 GNSS 卫星的在信号发射时刻的位置发生了变化，该现象也称作 Sagnac 效应。

![Geometric Range and Earth Rotation Correction](https://raw.githubusercontent.com/salmoshu/Winchell-ImgBed/main/img/20250219-234443.jpg)
<p style="text-align: center; font-family: 'Microsoft YaHei', SimSun, Arial, sans-serif; font-size: 14px;">图10-2 几何距离和地球自转校正</p>

地球自转引起的距离改正公式如下：
$$
\begin{array}{c}{\left[\begin{array}{l}x^{s^{\prime}} \\ y^{s^{s^{\prime}}} \\ z^{s^{\prime}}\end{array}\right]=\left[\begin{array}{ccc}\cos \omega \tau & \sin \omega \tau & 0 \\ -\sin \omega \tau & \cos \omega \tau & 0 \\ 0 & 0 & 1\end{array}\right]\left[\begin{array}{l}x^{s} \\ y^{s} \\ z^{s}\end{array}\right]} \\ \delta_{\text {sagnac, } r, j}=\frac{\omega_{e}}{c}\left(x^{s} y_{r}-y^{s} x_{r}\right)\end{array}
$$
上式中 $\left(x^{s^\prime}, y^{s^{\prime}}, z^{s^{\prime}}\right)$ 为地球自旋转后卫星的坐标值, $\left(x^{s}, y^{s}, z^{s}\right)$ 为地球自旋转前卫星的坐标值, $\omega_{e}$ 代表地球自传角速度值, $\tau$ 为卫星发射信号时刻到接收机接收卫星时刻的历元数。改正地球自转后的近似几何距离近似几何距离如下：
$$
\begin{array}{l} \rho_{r}^{s} \approx\left\|\boldsymbol{r}_{r}\left(t_{r}\right)-\boldsymbol{r}^{s}\left(t^{s}\right)\right\|+\frac{\omega_{e}}{c}\left(x^{s} y_{r}-y^{s} x_{r}\right) \end{array}
$$
接收机到卫星方向的观测矢量计算公式如下：
$$
\boldsymbol{e}_{r}^{s}=\frac{\boldsymbol{r}^{s}\left(t^{s}\right)-\boldsymbol{r}_{r}\left(t_{r}\right)}{\left\|\boldsymbol{r}^{s}\left(t^{s}\right)-\boldsymbol{r}_{r}\left(t_{r}\right)\right\|}
$$

**2. 参数列表**

```c
/* args */
double *rs       I   satellite position (ecef at transmission) (m)
double *rr       I   receiver position (ecef at reception) (m)
double *e        O   line-of-sight unit vector (ecef)
/* return */
double               geometric distance (m) (0>:error/no satellite position)
```

**3. 执行流程**

- 检查卫星到 WGS84 坐标系原点的距离是否大于基准椭球体的长半径。
- `ps-pr`，计算由接收机指向卫星方向的观测矢量，之后在计算出相应的单位矢量。
- 考虑到地球自转，即信号发射时刻卫星的位置与信号接收时刻卫星的位置在 WGS84 坐标系中并不一致，进行关于 Sagnac 效应的校正。

::: details 点击查看代码
```c
extern double geodist(const double *rs, const double *rr, double *e)
{
    double r;
    int i;
    
    if (norm(rs,3)<RE_WGS84) return -1.0; // 检查卫星到 WGS84坐标系原点的距离是否大于基准椭球体的长半径。
    for (i=0;i<3;i++) e[i]=rs[i]-rr[i];   // 求卫星和接收机坐标差e[]
    r=norm(e,3);                          // 求未经萨格纳克效应改正的距离
    for (i=0;i<3;i++) e[i]/=r;            // 接收机到卫星的单位向量e[] (E.3.9)
    return r+OMGE*(rs[0]*rr[1]-rs[1]*rr[0])/CLIGHT; // (E.3.8b)
}
```
:::

### 10.4.2 satazel()：计算方位角、高度角

**1. 基本原理**

![ Local Coordinates and Azimuth and Elevation Angles](https://raw.githubusercontent.com/salmoshu/Winchell-ImgBed/main/img/20250219-235235.jpg)
<p style="text-align: center; font-family: 'Microsoft YaHei', SimSun, Arial, sans-serif; font-size: 14px;">图10-3 局部坐标系与方位角和仰角</p>

方位角范围在 $[0,2 \pi]$，高度角范围在 $[-\frac{\pi}{2},\frac{\pi}{2}]$；以接收机为原点，建立站心坐标系 ENU，根据卫星 ENU 下方向矢量可以得到高度角、方位角，公式如下：
$$
\begin{array}{l}\boldsymbol{e}_{r, \text { enu }}^{s}=\boldsymbol{E}_{r} \boldsymbol{e}_{r}^{s}=\left(e_{e}, e_{n}, e_{u}\right)^{T} \\ A z_{r}^{s}=\operatorname{ATAN} 2\left(e_{e}, e_{n}\right) \\ E l_{r}^{s}=\arcsin \left(e_{u}\right)\end{array}
$$
对应的代码如下，传入接收机 LLH 坐标 `pos`、接收机到卫星方向的观测矢量 `e`，计算之后返回弧度制的高度角，如果传了参三 `azel`，那么 `azel[0]` 是方位角、`azel[1]` 是高度角。

**2. 参数列表**

```c
/* args */
double *pos      I   geodetic position {lat,lon,h} (rad,)
double *e        I   receiver-to-satellilte unit vevtor (ecef)
double *azel     IO  azimuth/elevation {az,el} (rad) (NULL: no output)  (0.0<=azel[0]<2*pi,-pi/2<=azel[1]<=pi/2)
/* return */
double               elevation angle (rad)
```

**3. 执行流程**
- 调用 `ecef2enu` 函数，将 `pos` 处的向量转换到以改点为原点的站心坐标系中。
- 调用 `atan2` 函数计算出方位角，然后再算出仰角。

**4. 注意事项**

- 这里在计算方位角时，要使用 `atan2` 函数，而不能是 `atan` 函数，详细原因见附录A.5。

::: details 点击查看代码
```c
extern double satazel(const double *pos, const double *e, double *azel)
{
    double az=0.0,el=PI/2.0,enu[3];
    
    if (pos[2]>-RE_WGS84) {
        ecef2enu(pos,e,enu);
        az=dot(enu,enu,2)<1E-12?0.0:atan2(enu[0],enu[1]);
        if (az<0.0) az+=2*PI;
        el=asin(enu[2]);
    }
    if (azel) {azel[0]=az; azel[1]=el;}
    return el;
}
```
:::

<!-- 

## 10.5 电离层处理

> 模型公式 [RTKLIB-Manual-CN E.5章节](/algorithm/RTKLIB-Manual-CN/09-appendixE-E.5.html)

### 10.5.1 概述

1. **基本理解**：GNSS 卫星电磁波信号在传播过程中需要穿过地球大气层，会产生一些大气延迟误差，包括电离层延迟误差和对流层延迟误差。

2. **电离层的特点**：
   - **分布**：电离层的范围从离地面约50公里开始一直伸展到约1000公里高度的地球高层大气空域。
   - **特性**：电离层的主要特性由电子密度、电子温度、碰撞频率、离子密度、离子温度和离子成分等空间分布的基本参数来表示。
   - **研究对象**：电离层研究的核心是**电子密度**随高度的分布。**电子密度**即单位体积内的自由电子数，其变化受**大气成分**、**大气密度**和**太阳辐射通量**等因素影响。**电子密度**由自由电子的**产生**、**消失**和**迁移**三种效应决定，不同区域这三种效应的作用方式和相对重要性存在差异。电离层中的自由电子和离子会使无线电波传播速度改变，引发**折射**、**反射**、**散射**、**极化面旋转**及不同程度的**吸收**。

3. **电离层延迟与电子总量关系**：电离层延迟与单位面积的横截面在信号传播路径上拦截的电子总量 $N_e$ 成正比，且与载波频率 $f$ 的平方成反比，弥散性的电离层降低了测距码的传播速度，而加快了载波相位的传播速度，如下所示。
   $$
   I=I_ \rho=-I_\phi=40.28\frac{N_e}{f^2}
   $$

4. **电离层模型假设**：电离层的三维结构导致其电子密度在水平和垂直方向上分布不均匀。GNSS信号的电离层延迟是整个传播路径上电子密度的积分，与信号的高度角和方位角相关。然而，GNSS用户更关注信号传播路径上的总电子含量（TEC），而非电子密度的具体分布。为了简化计算，GNSS数据处理中引入了**单层假设**，即将电离层压缩为一个高度为**H**的无限薄层，假设所有自由电子集中分布在此薄层上，进而对垂直总电子含量（VTEC）进行计算和处理。示意图如下： 

   <img src="https://raw.githubusercontent.com/salmoshu/Winchell-ImgBed/main/img/20250709-140753.jpg" style="width: 80%; margin: 0 auto;"/>

5. **常见方法或模型**：传统的双频测地型接收机通过**双频观测值无电离层组合**消除一阶电离层延迟误差，而单频数据则采用**电离层模型改正**来削弱误差影响。常用的电离层延迟改正模型包括**克罗布歇（Klobuchar）模型**、**格网（Global Ionosphere Maps，GIM）模型**和**国际参考电离层（International Reference Ionosphere，IRI）模型**。

### 10.5.2 ionocorr()：计算电离层延时

**1. 参数列表**

```c
/* args */
gtime_t  time      I   time
nav_t    *nav      I   navigation data
int      sat       I   satellite number
double   *pos      I   receiver position {lat,lon,h} (rad|m)
double   *azel     I   azimuth/elevation angle {az,el} (rad)
int      ionoopt   I   ionospheric correction option (IONOOPT_???)
double   *ion      O   ionospheric delay (L1) (m)
double   *var      O   ionospheric delay (L1) variance (m^2)
/* return */
int                    status (1:ok,0:error)
```

**2. 执行流程**

<img style="margin: 0 auto;" src="https://raw.githubusercontent.com/salmoshu/Winchell-ImgBed/main/img/20250709-135832.svg"/>

- 根据 `opt` 的值，选用不同的电离层模型计算方法。当 `ionoopt==IONOOPT_BRDC` 时，调用 `ionmodel` ，计算 Klobuchar 模型时的电离层延时 (L1，m)；当 `ionoopt==IONOOPT_TEC` 时，调用 `iontec` ，计算 TEC网格模型时的电离层延时 (L1，m)。

**3. 注意事项**

- 计算的是 L1 信号的电离层延时 I ，当使用其它频率信号时，依据所用信号频组中第一个频率的波长与 L1 波长的比例关系，对上一步得到的电离层延时进行修正。
- 当 `ionoopt==IONOOPT_IFLC` 时，此时通过此函数计算得到的延时和方差都为 0。其实，对于 IFLC 模型，其延时值在 `prange` 函数中计算伪距时已经包括在里面了，而方差是在 `varerr` 函数中计算的，并且会作为导航系统误差的一部分给出。

::: details 点击查看代码
```c
// ionocorr (pntpos.c)
extern int ionocorr(gtime_t time, const nav_t *nav, int sat, const double *pos,
                    const double *azel, int ionoopt, double *ion, double *var)
{
    int err=0;

    trace(4,"ionocorr: time=%s opt=%d sat=%2d pos=%.3f %.3f azel=%.3f %.3f\n",
          time_str(time,3),ionoopt,sat,pos[0]*R2D,pos[1]*R2D,azel[0]*R2D,
          azel[1]*R2D);
    
    /* SBAS ionosphere model */
    if (ionoopt==IONOOPT_SBAS) {
        if (sbsioncorr(time,nav,pos,azel,ion,var)) return 1;
        err=1;
    }
    /* IONEX TEC model */
    if (ionoopt==IONOOPT_TEC) {
        if (iontec(time,nav,pos,azel,1,ion,var)) return 1;
        err=1;
    }
    /* QZSS broadcast ionosphere model */
    if (ionoopt==IONOOPT_QZS&&norm(nav->ion_qzs,8)>0.0) {
        *ion=ionmodel(time,nav->ion_qzs,pos,azel);
        *var=SQR(*ion*ERR_BRDCI);
        return 1;
    }
    /* GPS broadcast ionosphere model */
    if (ionoopt==IONOOPT_BRDC||err==1) {
        *ion=ionmodel(time,nav->ion_gps,pos,azel);
        *var=SQR(*ion*ERR_BRDCI);
        return 1;
    }
    *ion=0.0;
    *var=ionoopt==IONOOPT_OFF?SQR(ERR_ION):0.0;
    return 1;
}
```
:::

### 10.5.3 ionmodel()：广播星历电离层改正

**1. 模型原理**

SPP 中使用克罗布歇模型计算 L1 的电离层改正量，将晚间的电离层时延视为常数，取值为 5ns，把白天的时延看成是余弦函数中正的部分。于是天顶方向调制在 L1 载波上的测距码的电离层时延可表示为：

$$
T_{g}=5 \times 10^{-9}+A \cos \frac{2 \pi}{P}\left(t-14^{h}\right)
$$
振幅 $A$ 和周期 $P$ 分别为：
$$
\begin{array}{l}
A=\sum_{i=0}^{3} \alpha_{i}\left(\varphi_{m}\right)^{i} \\
P=\sum_{i=0}^{3} \beta_{i}\left(\varphi_{m}\right)^{i}
\end{array}
$$
全球定位系统向单频接收机用户提供的电离层延迟改正时就采用上述模型。其中 $\alpha_i$ 和 $\beta_i$ 是地面控制系统根据该天为一年中的第几天（将一年分为 37 个区间）以及前 5 天太阳的平均辐射流量（共分10档）从 370 组常数中选取的，然后编入星导航电文播发给用户。卫星在其播发的导航电文中提供了这 8 个电离层延迟参数：
$$
p_{\text {ion }}=\left(\alpha_{0}, \alpha_{1}, \alpha_{2}, \alpha_{3}, \beta_{0}, \beta_{1}, \beta_{2}, \beta_{3}\right)^{T}
$$
根据根据参数 $\alpha_0，\alpha_1，\alpha_2，\alpha_3$ 确定振幅 $A$，根据根据参数 $\beta_0，\beta_1，\beta_2，\beta_3$ 确定周期 $T$，再给定一个以秒为单位的当地时间 $t$，就能算出**天顶电离层延迟**。再由天顶对流层延迟根据倾斜率转为卫星方向对流层延迟。

电离层分布在离地面 60-1000km 的区域内。当卫星不在测站的天顶时，信号传播路径上每点的地方时和纬度均不相同，为了简化计算，我们将整个电离层压缩为一个单层，将整个电离层中的自由电子都集中在该单层上，用它来代替整个电离层。这个电离层就称为中心电离层。中心电离层离地面的高度通常取 350km。式中的参数 $t$ 和式中的参数 $\varphi_{m}$ 分别为卫星言号传播路径与中心电离层的交点 $P^{\prime}$ 的时角和地磁纬度，因为只有 $P^{\prime}$ 才能反映卫星信号所受到的电离层延迟的总的情况。

综上，已知大地经度、 大地纬度、卫星的高度角和卫星测站的方位角，电离层延迟计算方法如下：

计算测站 $P$ 和 $P^{\prime}$ 在地心的夹角：
$$
\psi=0.0137 /(E l+0.11)-0.022
$$
计算交点 $P^{\prime}$ 的地心纬度：
$$
\varphi_{i}=\varphi+\psi \cos A z
$$

$$
\left\{\begin{array}{ll}\varphi_{l}>+0.416 & \varphi_{l}=+0.416 \\ \varphi{l}>-0.416 & \varphi_{l}=-0.416\end{array}\right.
$$

计算交点 $P^{\prime}$ 的地心经度：
$$
\lambda_{i}=\lambda+\psi \sin A z / \cos \varphi_{i}
$$
计算地磁纬度：
$$
\varphi_{m}=\varphi_{i}+0.064 \cos \left(\lambda_{i}-1.617\right)
$$
计算观测瞬间交点 $P^{\prime}$ 处的地方时：
$$
  t=4.32 \times 10^{4} \lambda_{i}+t 
$$

$$
\left\{\begin{array}{ll}t>86400 & t=t-86400 \\ t<0 & t=t+86400\end{array}\right.
$$

计算倾斜因子：
$$
 F=1.0+16.0 \times(0.53-E l)^{3}
$$
计算电离层时间延迟：
$$
\begin{array}{l}x=2 \pi(t-50400) / \sum_{n=0}^{3} \beta_{n} \varphi_{m}{ }^{n} \\ I_{r}^{s}=\left\{\begin{array}{cc}F \times 5 \times 10^{-9} \\ F \times\left(5 \times 10^{-9}+\sum_{n=1}^{4} \alpha_{n} \varphi_{m}{ }^{n} \times\left(1-\frac{x^{2}}{2}+\frac{x^{4}}{24}\right)\right) & (|x|>1.57)\end{array}\right.\end{array}
$$

计算的是 L1 信号的电离层延时 I ，当使用其它频率信号时，依据所用信号频组中第一个频率的波长与 L1 波长的比例关系，对上一步得到的电离层延时进行修正，不考虑模糊度情况下改正公式为：
$$
I=\frac{\Phi_{2}-\Phi_{1}}{1-\left(f_{1} / f_{2}\right)^{2}}
$$

**2. 参数列表**

```c
/* args */
gtime_t t        I   time (gpst)
double *ion      I   iono model parameters {a0,a1,a2,a3,b0,b1,b2,b3}
double *pos      I   receiver position {lat,lon,h} (rad,m)
double *azel     I   azimuth/elevation angle {az,el} (rad)
/* return */
double               ionospheric delay (L1) (m)
```

**3. 注意事项**

- 主要都是数学计算，其过程可以在 ICD-GPS-200C P148 [15]中找到；
- 这里计算的电离层延时是相对于 GPS-L1 信号而言的，其它频率信号需要进行一次转换。
- 计算过程中很多角度的单位是半圆，即 $\pi$ 弧度。在阅读代码时，记住这一点非常重要！比如，虽然上述过程与 ICD-GPS-200C P148 中一致，但可能与大部分资料上的过程还是会有所区别。尤其是以下公式：
  - $ψ = \frac{0.0137}{E+0.01} - 0.022$ （参考文献[16]）
  - $EA = \frac{445°}{el+20°} - 4^\circ$ （参考文献[15]）
  
  但是将下面公式的角度转化成半圆，即左右两边都除以 180，就可以得到上面的公式了！

::: details 点击查看代码
```c
extern double ionmodel(gtime_t t, const double *ion, const double *pos,
                       const double *azel)
{
    const double ion_default[]={ /* 2004/1/1 */
        0.1118E-07,-0.7451E-08,-0.5961E-07, 0.1192E-06,
        0.1167E+06,-0.2294E+06,-0.1311E+06, 0.1049E+07
    };
    double tt,f,psi,phi,lam,amp,per,x;
    int week;
    
    if (pos[2]<-1E3||azel[1]<=0) return 0.0;
    if (norm(ion,8)<=0.0) ion=ion_default;  //若没有电离层参数，用默认参数


    /* earth centered angle (semi-circle) */    //地球中心角
    psi=0.0137/(azel[1]/PI+0.11)-0.022;         //计算地心角(E.5.6)
    
    /* subionospheric latitude/longitude (semi-circle) */   
    phi=pos[0]/PI+psi*cos(azel[0]);                 //计算穿刺点地理纬度(E.5.7)
    if      (phi> 0.416) phi= 0.416;        //phi不超出(-0.416,0.416)范围
    else if (phi<-0.416) phi=-0.416;
    lam=pos[1]/PI+psi*sin(azel[0])/cos(phi*PI);     //计算穿刺点地理经度(E.5.8)
    
    /* geomagnetic latitude (semi-circle) */
    phi+=0.064*cos((lam-1.617)*PI);                 //计算穿刺点地磁纬度(E.5.9)
    
    /* local time (s) */
    tt=43200.0*lam+time2gpst(t,&week);              //计算穿刺点地方时(E.5.10)
    tt-=floor(tt/86400.0)*86400.0; /* 0<=tt<86400 */
    
    /* slant factor */
    f=1.0+16.0*pow(0.53-azel[1]/PI,3.0);            //计算投影系数(E.5.11)
    
    /* ionospheric delay */
    amp=ion[0]+phi*(ion[1]+phi*(ion[2]+phi*ion[3]));
    per=ion[4]+phi*(ion[5]+phi*(ion[6]+phi*ion[7]));
    amp=amp<    0.0?    0.0:amp;
    per=per<72000.0?72000.0:per;
    x=2.0*PI*(tt-50400.0)/per;                      //(E.5.12)
    
    return CLIGHT*f*(fabs(x)<1.57?5E-9+amp*(1.0+x*x*(-0.5+x*x/24.0)):5E-9);     //(E.5.13)
}
```
:::



### 10.5.4 readtec()：TEC读取入口函数

<img style="margin: 10px auto; width: 70%;" src="https://raw.githubusercontent.com/salmoshu/Winchell-ImgBed/main/img/20250710-235356.jpg"/>
<p style="text-align: center; font-family: 'Microsoft YaHei', SimSun, Arial, sans-serif; font-size: 14px;">图10.5-3 readtec 函数调用关系</p>

该部分参考文献[17]。

**1. 参数列表**

```c
/* args */
char   *file       I   ionex tec grid file
                       (wind-card * is expanded)
nav_t  *nav        IO  navigation data
                       nav->nt, nav->ntmax and nav->tec are modified
int    opt         I   read option (1: no clear of tec data,0:clear)
/* return */
none
```

**2. 调用函数**

* **readionexh**()：读取文件头，循环读取每一行，根据注释读取前面内容，如果遇到`START OF AUX DATA` ，调用readionexdcb()读取DCB参数。
* **readionexdcb**()：循环读取DCB参数和对应的均方根误差，直到`END OF AUS DATA`
* **readionexb**()：循环读取TEC格网数据和均方根误差，`type`为1是TEC格网，`type`为2是均分根误差，调用`addtec()`将tec格网数据存到`nav->tec[]`
* **combtec**()：合并`nav->tec[]`中时间相同的项。

**3. 执行流程**

* 开辟内存空间
* 扩展`file`的*到`efile[]` ，遍历`efile[] `：
* `fopen()`以读的方式打开
* 调用`readionexh()`读ionex文件头 
* 调用`readionexb()`读ionex文件体 ，其中会调用`addtec()`将tec格网数据存到`nav->tec[]`
* 读取完之后，调用`combtec()`合并tec格网数据
* 存DCB参数到`nav->cbias`

::: details 点击查看代码
```c
extern void readtec(const char *file, nav_t *nav, int opt)
{
    FILE *fp;
    double lats[3]={0},lons[3]={0},hgts[3]={0},rb=0.0,nexp=-1.0;
    double dcb[MAXSAT]={0},rms[MAXSAT]={0};
    int i,n;
    char *efiles[MAXEXFILE];
    
    trace(3,"readtec : file=%s\n",file);
    
    /* clear of tec grid data option */
    if (!opt) { //如果没有opt，释放nav->tec，nav->ntmax置0
        free(nav->tec); nav->tec=NULL; nav->nt=nav->ntmax=0;
    }
    for (i=0;i<MAXEXFILE;i++) {     //为efile[]开辟空间
        if (!(efiles[i]=(char *)malloc(1024))) {
            for (i--;i>=0;i--) free(efiles[i]);
            return;
        }
    }
    /* expand wild card in file path */
    n=expath(file,efiles,MAXEXFILE);    //扩展file的*到efile[]
    
    //遍历efile[]
    for (i=0;i<n;i++) {
        if (!(fp=fopen(efiles[i],"r"))) {       //以读的方式打开
            trace(2,"ionex file open error %s\n",efiles[i]);
            continue;
        }
        /* read ionex header */     //调用readionexh()读ionex文件头
        if (readionexh(fp,lats,lons,hgts,&rb,&nexp,dcb,rms)<=0.0) {     
            trace(2,"ionex file format error %s\n",efiles[i]);
            continue;
        }
        /* read ionex body */       //调用readionexb()读ionex文件体
        readionexb(fp,lats,lons,hgts,rb,nexp,nav);
        
        fclose(fp);
    }
    for (i=0;i<MAXEXFILE;i++) free(efiles[i]);
    
    /* combine tec grid data */
    if (nav->nt>0) combtec(nav);    //调用combtec()合并tec格网数据
    
    /* P1-P2 dcb */
    for (i=0;i<MAXSAT;i++) {
        nav->cbias[i][0]=CLIGHT*dcb[i]*1E-9; /* ns->m */	//存DCB到nav->cbias
    }
}
```
:::

### 10.5.5 iontec()：TEC格网改正主入口函数

![](https://pic-bed-1316053657.cos.ap-nanjing.myqcloud.com/img/4b6779544615489cbcadaabd59c74d6d.png)

##### 1.原理
格网改正模型电离层格网模型文件 IONEX，通过内插获得穿刺点位置，并结合当天电离层格网数据求出穿刺点的垂直电子含量，获得电离层延迟误差 。

##### 2.iontec()：TEC格网改正主入口函数

由所属时间段两端端点的TEC网格数据**时间插值**计算出电离层延时 (L1) (m) 

* 检测高度角和接收机高度是否大于阈值 
* 从 `nav_t.tec`中找出第一个`tec[i].time`>`time`信号接收时间 
* 确保 time是在所给出的`nav_t.tec`包含的时间段之中 ，通过确认所找到的时间段的右端点减去左端点，来确保时间间隔不为0 
* 调用`iondelay()`来计算所属时间段两端端点的电离层延时
* 由两端的延时，插值计算出观测时间点处的值 

```c
extern int iontec(gtime_t time, const nav_t *nav, const double *pos,
                 const double *azel, int opt, double *delay, double *var)
{
   double dels[2],vars[2],a,tt;
   int i,stat[2];
   
   
   trace(3,"iontec  : time=%s pos=%.1f %.1f azel=%.1f %.1f\n",time_str(time,0),
         pos[0]*R2D,pos[1]*R2D,azel[0]*R2D,azel[1]*R2D);
   
   if (azel[1]<MIN_EL||pos[2]<MIN_HGT) {   //检测高度角和接收机高度是否大于阈值
       *delay=0.0;
       *var=VAR_NOTEC;
       return 1;
   }
   for (i=0;i<nav->nt;i++) {       //从 nav_t.tec中找出第一个tec[i].time>time信号接收时间
       if (timediff(nav->tec[i].time,time)>0.0) break;
   }
   if (i==0||i>=nav->nt) {     //确保 time是在所给出的nav_t.tec包含的时间段之中
       trace(2,"%s: tec grid out of period\n",time_str(time,0));
       return 0;
   }
   if ((tt=timediff(nav->tec[i].time,nav->tec[i-1].time))==0.0) {  //通过确认所找到的时间段的右端点减去左端点，来确保时间间隔 != 0
       trace(2,"tec grid time interval error\n");
       return 0;
   }
   /* ionospheric delay by tec grid data */    //调用 iondelay来计算所属时间段两端端点的电离层延时
   stat[0]=iondelay(time,nav->tec+i-1,pos,azel,opt,dels  ,vars  );
   stat[1]=iondelay(time,nav->tec+i  ,pos,azel,opt,dels+1,vars+1);
   
   //由两端的延时，插值计算出观测时间点处的值
   if (!stat[0]&&!stat[1]) {   //两个端点都计算出错，输出错误信息，返回 0
       trace(2,"%s: tec grid out of area pos=%6.2f %7.2f azel=%6.1f %5.1f\n",
             time_str(time,0),pos[0]*R2D,pos[1]*R2D,azel[0]*R2D,azel[1]*R2D);
       return 0;
   }
   if (stat[0]&&stat[1]) { /* linear interpolation by time */  //两个端点都有值，线性插值出观测时间点的值，返回 1
       a=timediff(time,nav->tec[i-1].time)/tt;
       *delay=dels[0]*(1.0-a)+dels[1]*a;
       *var  =vars[0]*(1.0-a)+vars[1]*a;
   }
   else if (stat[0]) { /* nearest-neighbour extrapolation by time */   //只有一个端点有值，将其结果作为观测时间处的值，返回 1
       *delay=dels[0];
       *var  =vars[0];
   }
   else {
       *delay=dels[1];
       *var  =vars[1];
   }
   trace(3,"iontec  : delay=%5.2f std=%5.2f\n",*delay,sqrt(*var));
   return 1;
}
```

### 10.5.6 iondelay()：计算指定时间电离层延时 (L1) (m)

* while大循环`tec->ndata[2]`次：
* 调用`ionppp()`函数，计算当前电离层高度，穿刺点的位置 {lat,lon,h} (rad,m)和倾斜率
* 按照`M-SLM`映射函数重新计算倾斜率 
* 在日固系中考虑地球自转，重新计算穿刺点经度 
* 调用`interptec()`格网插值获取`vtec `
* `*delay+=fact*fs*vtec `,`*var+=fact*fact*fs*fs*rms*rms `

```c
static int iondelay(gtime_t time, const tec_t *tec, const double *pos,
                   const double *azel, int opt, double *delay, double *var)
{
   const double fact=40.30E16/FREQ1/FREQ1; /* tecu->L1 iono (m) */
   double fs,posp[3]={0},vtec,rms,hion,rp;
   int i;
   
   trace(3,"iondelay: time=%s pos=%.1f %.1f azel=%.1f %.1f\n",time_str(time,0),
         pos[0]*R2D,pos[1]*R2D,azel[0]*R2D,azel[1]*R2D);
   
   *delay=*var=0.0;
   
   //opt 模式选项   bit0: 0:earth-fixed,1:sun-fixed
   //              bit1: 0:single-layer,1:modified single-layer
   
   for (i=0;i<tec->ndata[2];i++) { /* for a layer */
       
       hion=tec->hgts[0]+tec->hgts[2]*i;
       
       //调用ionppp()函数，计算当前电离层高度，穿刺点的位置 {lat,lon,h} (rad,m)和倾斜率
       /* ionospheric pierce point position */
       fs=ionppp(pos,azel,tec->rb,hion,posp);  
       
       if (opt&2) {    //按照M-SLM映射函数重新计算倾斜率
           /* modified single layer mapping function (M-SLM) ref [2] */
           rp=tec->rb/(tec->rb+hion)*sin(0.9782*(PI/2.0-azel[1]));
           fs=1.0/sqrt(1.0-rp*rp);
       }
       if (opt&1) {    //在日固系中考虑地球自转，重新计算穿刺点经度
           /* earth rotation correction (sun-fixed coordinate) */
           posp[1]+=2.0*PI*timediff(time,tec->time)/86400.0;
       }
       /* interpolate tec grid data */     //格网插值获取vtec
       if (!interptec(tec,i,posp,&vtec,&rms)) return 0;
       
       *delay+=fact*fs*vtec;
       *var+=fact*fact*fs*fs*rms*rms;
   }
   trace(4,"iondelay: delay=%7.2f std=%6.2f\n",*delay,sqrt(*var));
   
   return 1;
}
```

### 10.5.7 ionppp()：计算电离层穿刺点

位置 {lat,lon,h} (rad,m)和倾斜率做返回值 

![](https://pic-bed-1316053657.cos.ap-nanjing.myqcloud.com/img/e5f8c8b484e2412e86ff316d90d08702.png)


```c
extern double ionppp(const double *pos, const double *azel, double re,
                    double hion, double *posp)
{
   double cosaz,rp,ap,sinap,tanap;
   
   rp=re/(re+hion)*cos(azel[1]);   //(E.5.15)
   // z并不是仰角azel[1]，而是仰角关于的补角，所以程序中在计算 rp是采用的是 cos(azel[1])的写法
   ap=PI/2.0-azel[1]-asin(rp);     //(E.5.14)(E.5.16)
   sinap=sin(ap);
   tanap=tan(ap);
   cosaz=cos(azel[0]);
   posp[0]=asin(sin(pos[0])*cos(ap)+cos(pos[0])*sinap*cosaz);  //(E.5.17)
   
   if ((pos[0]> 70.0*D2R&& tanap*cosaz>tan(PI/2.0-pos[0]))||
       (pos[0]<-70.0*D2R&&-tanap*cosaz>tan(PI/2.0+pos[0]))) {
       posp[1]=pos[1]+PI-asin(sinap*sin(azel[0])/cos(posp[0]));    //(E.5.18a)
   }       
   else {
       posp[1]=pos[1]+asin(sinap*sin(azel[0])/cos(posp[0]));       //(E.5.18b)
   }
   return 1.0/sqrt(1.0-rp*rp);     //返回倾斜率

   //可能因为后面再从 TEC网格数据中插值时，并不需要高度信息，所以这里穿刺点位置posp[2]没有赋值
}
```

### 10.5.8 dataindex()：获取TEC格网数据下标

先判断点位是否在格网中，之后获取网格点的tec数据在 tec.data中的下标

```c
static int dataindex(int i, int j, int k, const int *ndata) //(i:lat,j:lon,k:hgt)
{
   if (i<0||ndata[0]<=i||j<0||ndata[1]<=j||k<0||ndata[2]<=k) return -1;
   return i+ndata[0]*(j+ndata[1]*k);
}
```

### 10.5.9 interptec()：插值计算穿刺点处TEC

通过在经纬度网格点上进行双线性插值，计算第k个高度层时穿刺点处的电子数总量TEC

![](https://pic-bed-1316053657.cos.ap-nanjing.myqcloud.com/img/8c24cde54f6e4005b7858f2fef0c1dce.png)


```c
/* interpolate tec grid data -------------------------------------------- -----
* args:tec_t *tec      I   tec grid data
*      int k           I   高度方向上的序号，可以理解为电离层序号
*      double *posp    I   pierce point position {lat,lon,h} (rad,m)
*      double *value   O   计算得到的刺穿点处的电子数总量(tecu)
*      double *rms     O   所计算的电子数总量的误差的标准差(tecu)
---------------------------------------------------------------------------- */
static int interptec(const tec_t *tec, int k, const double *posp, double *value,
                    double *rms)
{
   double dlat,dlon,a,b,d[4]={0},r[4]={0};
   int i,j,n,index;
   
   trace(3,"interptec: k=%d posp=%.2f %.2f\n",k,posp[0]*R2D,posp[1]*R2D);
   *value=*rms=0.0;        // 将 value和 rms所指向的值置为 0
   
   if (tec->lats[2]==0.0||tec->lons[2]==0.0) return 0; //检验 tec的纬度和经度间隔是否为 0。是，则直接返回 0
   
   // 将穿刺点的经纬度分别减去网格点的起始经纬度，再除以网格点间距，对结果进行取整，
   // 得到穿刺点所在网格的序号和穿刺点所在网格的位置(比例) i,j 
   dlat=posp[0]*R2D-tec->lats[0];
   dlon=posp[1]*R2D-tec->lons[0];
   if (tec->lons[2]>0.0) dlon-=floor( dlon/360)*360.0; /*  0<=dlon<360 */
   else                  dlon+=floor(-dlon/360)*360.0; /* -360<dlon<=0 */
   a=dlat/tec->lats[2];
   b=dlon/tec->lons[2];
   i=(int)floor(a); a-=i;
   j=(int)floor(b); b-=j;
   
   // 调用 dataindex() 函数分别计算这些网格点的 tec 数据在 tec.data中的下标，
   // 按从左下到右上的顺序
   // 从而得到这些网格点处的 TEC 值和相应误差的标准差
   /* get gridded tec data */
   for (n=0;n<4;n++) {     
       if ((index=dataindex(i+(n%2),j+(n<2?0:1),k,tec->ndata))<0) continue;
       d[n]=tec->data[index];
       r[n]=tec->rms [index];
   }
   if (d[0]>0.0&&d[1]>0.0&&d[2]>0.0&&d[3]>0.0) {
       // 穿刺点位于网格内，使用双线性插值计算出穿刺点的 TEC 值
       /* bilinear interpolation (inside of grid) */
       *value=(1.0-a)*(1.0-b)*d[0]+a*(1.0-b)*d[1]+(1.0-a)*b*d[2]+a*b*d[3];
       *rms  =(1.0-a)*(1.0-b)*r[0]+a*(1.0-b)*r[1]+(1.0-a)*b*r[2]+a*b*r[3];
   }
   // 穿刺点不位于网格内,使用最邻近的网格点值作为穿刺点的 TEC值，不过前提是网格点的 TEC>0
   /* nearest-neighbour extrapolation (outside of grid) */
   else if (a<=0.5&&b<=0.5&&d[0]>0.0) {*value=d[0]; *rms=r[0];}
   else if (a> 0.5&&b<=0.5&&d[1]>0.0) {*value=d[1]; *rms=r[1];}
   else if (a<=0.5&&b> 0.5&&d[2]>0.0) {*value=d[2]; *rms=r[2];}
   else if (a> 0.5&&b> 0.5&&d[3]>0.0) {*value=d[3]; *rms=r[3];}
   // 否则，选用四个网格点中 >0的值的平均值作为穿刺点的 TEC值
   else {
       i=0;
       for (n=0;n<4;n++) if (d[n]>0.0) {i++; *value+=d[n]; *rms+=r[n];}
       if(i==0) return 0;
       *value/=i; *rms/=i;
   }
   return 1;
}
```

## 10.6 对流层处理

### 10.6.1 对流层延迟改正概述

1. 对流层延迟一般指非电离大气对电磁波的折射，折射效应大部分发生在对流层，因此称
   为对流层延迟误差。对流层延迟影响与信号高度角有关，天顶方向影响能够达到2.3m，而高
   度角较小时，其影响量可达20m

2. 对流层可视为非弥散介质，其折射率$n$与电磁波频率$f$无关，于是GPS信号的群速率和相速率在对流层相等，无法使用类似消电离层的方法消除。

3. 对流层延迟可以分为干延迟和湿延迟两部分，总的对流层延迟可以根据天顶方向干湿延
   迟分量及其投影系数确定：
   $$
   d_{trop}=d_{zpd}M_d(\theta)+d_{zpw}M_w(\theta)
   $$

### 10.6.2 tropcorr()：根据选项调用对应函数计算对流层延迟T 

在rescode()中被调用，调用tropmodel()、sbstropcorr() 根据选项，计算对流层延迟T 

```c
extern int tropcorr(gtime_t time, const nav_t *nav, const double *pos,
                    const double *azel, int tropopt, double *trp, double *var)
{
    trace(4,"tropcorr: time=%s opt=%d pos=%.3f %.3f azel=%.3f %.3f\n",
          time_str(time,3),tropopt,pos[0]*R2D,pos[1]*R2D,azel[0]*R2D,
          azel[1]*R2D);
    
    /* Saastamoinen model */
    if (tropopt==TROPOPT_SAAS||tropopt==TROPOPT_EST||tropopt==TROPOPT_ESTG) {
        *trp=tropmodel(time,pos,azel,REL_HUMI);
        *var=SQR(ERR_SAAS/(sin(azel[1])+0.1));
        return 1;
    }
    /* SBAS (MOPS) troposphere model */
    if (tropopt==TROPOPT_SBAS) {
        *trp=sbstropcorr(time,pos,azel,var);
        return 1;
    }
    /* no correction */
    *trp=0.0;
    *var=tropopt==TROPOPT_OFF?SQR(ERR_TROP):0.0;
    return 1;
}
```
### 10.6.3 tropmodel()：Saastamoinen对流层模型改正 

![](https://pic-bed-1316053657.cos.ap-nanjing.myqcloud.com/img/708bfc5a018f4205a9551c56a44ec552.png)


```c
extern double tropmodel(gtime_t time, const double *pos, const double *azel,
                        double humi)
{
    const double temp0=15.0; /* temparature at sea level */
    double hgt,pres,temp,e,z,trph,trpw;
    
    if (pos[2]<-100.0||1E4<pos[2]||azel[1]<=0) return 0.0;
    
    /* standard atmosphere */
    hgt=pos[2]<0.0?0.0:pos[2];
    
    pres=1013.25*pow(1.0-2.2557E-5*hgt,5.2568);         // 求大气压P (E.5.1)
    temp=temp0-6.5E-3*hgt+273.16;                       // 求温度temp (E.5.2)
    e=6.108*humi*exp((17.15*temp-4684.0)/(temp-38.45)); // 求大气水汽压力e (E.5.3)
    
    /* saastamoninen model */ 
    z=PI/2.0-azel[1];       // 求天顶角z 卫星高度角azel[1]的余角
    trph=0.0022768*pres/(1.0-0.00266*cos(2.0*pos[0])-0.00028*hgt/1E3)/cos(z);   //求静力学延迟Th
    trpw=0.002277*(1255.0/temp+0.05)*e/cos(z);              //求湿延迟Tw (E.5.4)
    return trph+trpw;       // Saastamoinen中对流层延迟为静力学延迟Th湿延迟Tw的和
}
``` 
-->

<!-- 

**1. tec_t结构体：存tec格网数据**

```c
typedef struct {        /* TEC grid type */
    gtime_t time;       /* epoch time (GPST) */
    int ndata[3];       /* TEC grid data size {nlat,nlon,nhgt} */
    double rb;          /* earth radius (km) */
    double lats[3];     /* latitude start/interval (deg) */
    double lons[3];     /* longitude start/interval (deg) */
    double hgts[3];     /* heights start/interval (km) */
    double *data;       /* TEC grid data (tecu) */
    float *rms;         /* RMS values (tecu) */
} tec_t;
``` 

* 其存储在`nav_t`结构体`tec`中 ，`nt`表示tec数量
-->
