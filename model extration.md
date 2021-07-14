## 通过分析或查询的方式提高效率？

- 首先对于该问题新的理解：

  参考密码学中的线性分析和**差分分析**

  ​        差分分析：利用S盒的不均匀差分特征

  差分分析流程：

  <img src="https://wxw-typora-pics.oss-cn-chengdu.aliyuncs.com/img/D012270DF3CB664E34C877DEE52D9B13.png" alt="img" style="zoom: 33%;" />

  利用选择明文攻击的方式，通过构造差分链得到u<sup>4</sup>的差分特征，枚举第五轮的密钥，看看哪一个密钥下选择的明密文对符合差分特征的最多，那么合理猜测该密钥就是正确的密钥

  

  对比一下我们研究的差分攻击，我们研究的神经网络与SPN网络有很多相似的地方：

  1. S盒相当于neuron ，文中的差分攻击本质是利用ReLU函数在零点附近差分值的不同而构造的

  2. P盒+轮密钥相当于神经网络中的参数，即我们要恢复的
     $$
     
     \mathrm{A}^{\left( \mathrm{i} \right)}\mathrm{x}+\mathrm{b}^{\left( \mathrm{i} \right)}
     
     $$
     

  

