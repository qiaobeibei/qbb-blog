---
title: 强化学习PPO篇——卫星博弈
date: 2024-11-03 11:56:16
categories:
- 航天
- 强化学习
tags: 
- 强化学习
- 航天器
typora-root-url: ./..
---

##  1. PPO算法介绍

PPO算法通过使用剪辑的目标函数和多次小步更新，保证了策略更新的稳定性和样本效率。它适合处理高维连续动作空间的问题，并且在实践中表现出较好的稳定性和收敛速度。

![img](/images/$%7Bfiilename%7D/50e093d980624af1ab1eeeaa6f86e960.png)编辑

### 1）策略更新的目标函数

- PPO的核心是通过优化一个剪辑的目标函数来更新策略。
- 目标函数： LCLIP(θ)=Et[min(rt(θ)A^t,clip(rt(θ),1−ϵ,1+ϵ)A^t)] 
  - rt(θ)=πθ(at|st)πθold(at|st) 是当前策略和旧策略的概率比
  - A^t 是优势函数的估计值
  - ϵ 是一个小的超参数，控制策略更新的范围
- 通过最小化被剪辑后的目标函数，PPO限制了策略更新的幅度，避免过大的更新导致不稳定。

### 2） 优势函数估计

- PPO算法使用优势函数 A^t 来度量在状态 st 下采取动作 at 的好坏。常用的优势函数估计方法有GAE（Generalized Advantage Estimation）。
- 计算公式： A^t=∑l=0∞(γλ)lδtV

其中 δtV=rt+γV(st+1)−V(st) ， γ 是折扣因子， λ 是GAE的衰减系数。

### 3）多次小步更新

- PPO不像传统的策略梯度方法进行大步更新，而是通过多次小步更新来提高策略。这样可以在保持策略更新稳定的同时提高样本效率。
- 每次更新时，PPO从旧策略 θold 开始，以一定次数的迭代更新策略参数 θ ，确保在小范围内改进策略。

### 4）价值网络更新

- PPO通常使用一个价值函数 \( V(s_t; \phi) \) 来估计每个状态的价值，这个函数是通过最小化均方误差（MSE）来训练的： LVF(ϕ)=12Et[(V(st;ϕ)−Vttarget)2]
- 通过同时更新策略网络和价值网络，PPO可以更好地利用当前策略下的估计值，提高训练效率。

### 5）随机小批量优化

PPO在更新过程中通常采用随机小批量（mini-batch）梯度下降的方法。通过对每个小批量的数据进行优化，可以更有效地利用经验，提高训练速度。

### 6）经验回放

PPO在训练过程中使用每个时间步的数据进行优化，但不会像DQN那样使用经验回放。它在每个时间步收集数据并使用这些数据直接优化目标函数，以保证策略更新的一致性。

## 2. 环境

- python3.8.5
- 若干库

## 3. 代码

### 1）主函数

主函数中负责执行追捕者和逃逸者的训练函数，并用于测试。

```cpp
import matplotlib.pyplot as plt
import numpy as np
from normalization import Normalization, RewardScaling
from replaybuffer import ReplayBuffer
from ppo_continuous import PPO_continuous
from environment import satellites
from tqdm import tqdm
import plot_function as pf


# 先定义一个参数类，用来储存超参数
class args_param(object):
    def __init__(self, max_train_steps=int(3e6),
                 evaluate_freq=5e3,
                 save_freq=20,
                 policy_dist="Gaussian",
                 batch_size=2048,
                 mini_batch_size=64,
                 hidden_width=256,
                 hidden_width2 =128,
                 lr_a=0.0002,
                 lr_c=0.0002,
                 gamma=0.99,
                 lamda=0.95,
                 epsilon=0.1,
                 K_epochs=10,
                 max_episode_steps=1000,
                 use_adv_norm=True,
                 use_state_norm=True,
                 use_reward_norm=False,
                 use_reward_scaling=True,
                 entropy_coef=0.01,
                 use_lr_decay=True,
                 use_grad_clip=True,
                 use_orthogonal_init=True,
                 set_adam_eps=True,
                 use_tanh=True,
                 chkpt_dir="/mnt/datab/home/yuanwenzheng/PICTURE1"):
        self.max_train_steps = max_train_steps
        self.evaluate_freq = evaluate_freq
        self.save_freq = save_freq
        self.policy_dist = policy_dist
        self.batch_size = batch_size
        self.mini_batch_size = mini_batch_size
        self.hidden_width = hidden_width
        self.hidden_width2 = hidden_width2
        self.lr_a = lr_a
        self.lr_c = lr_c
        self.gamma = gamma
        self.lamda = lamda
        self.epsilon = epsilon
        self.K_epochs = K_epochs
        self.use_adv_norm = use_adv_norm
        self.use_state_norm = use_state_norm
        self.use_reward_norm = use_reward_norm
        self.use_reward_scaling = use_reward_scaling
        self.entropy_coef = entropy_coef
        self.use_lr_decay = use_lr_decay
        self.use_grad_clip = use_grad_clip
        self.use_orthogonal_init = use_orthogonal_init
        self.set_adam_eps = set_adam_eps
        self.use_tanh = use_tanh
        self.max_episode_steps = max_episode_steps
        self.chkpt_dir = chkpt_dir

    def print_information(self):
        print("Maximum number of training steps:", self.max_train_steps)
        print("Evaluate the policy every 'evaluate_freq' steps:", self.evaluate_freq)
        print("Save frequency:", self.save_freq)
        print("Beta or Gaussian:", self.policy_dist)
        print("Batch size:", self.batch_size)
        print("Minibatch size:", self.mini_batch_size)
        print("The number of neurons in hidden layers of the neural network:", self.hidden_width)
        print("Learning rate of actor:", self.lr_a)
        print("Learning rate of critic:", self.lr_c)
        print("Discount factor:", self.gamma)
        print("GAE parameter:", self.lamda)
        print("PPO clip parameter:", self.epsilon)
        print("PPO parameter:", self.K_epochs)
        print("Trick 1:advantage normalization:", self.use_adv_norm)
        print("Trick 2:state normalization:", self.use_state_norm)
        print("Trick 3:reward normalization:", self.use_reward_norm)
        print("Trick 4:reward scaling:", self.use_reward_scaling)
        print("Trick 5: policy entropy:", self.entropy_coef)
        print("Trick 6:learning rate Decay:", self.use_lr_decay)
        print("Trick 7: Gradient clip:", self.use_grad_clip)
        print("Trick 8: orthogonal initialization:", self.use_orthogonal_init)
        print("Trick 9: set Adam epsilon=1e-5:", self.set_adam_eps)
        print("Trick 10: tanh activation function:", self.use_tanh)


# 下面函数用来训练网络
def train_network(args, env, show_picture=True, pre_train=False, d_capture=0):
    epsiode_rewards = []
    epsiode_mean_rewards = []
    # 下面将导入env环境参数
    env.d_capture = d_capture
    args.state_dim = env.observation_space.shape[0]
    args.action_dim = env.action_space.shape[0]
    args.max_action = float(env.action_space[0][1])
    # 下面将定义一个缓冲区
    replay_buffer = ReplayBuffer(args)
    # 下面将定义PPO智能体类
    pursuer_agent = PPO_continuous(args, 'pursuer')
    evader_agent = PPO_continuous(args, 'evader')
    if pre_train:
        pursuer_agent.load_checkpoint()
    # 下面开始进行训练过程
    pbar = tqdm(range(args.max_train_steps), desc="Training of pursuer", unit="episode")
    for epsiode in pbar:
        # 每个回合首先对值进行初始化
        epsiode_reward = 0.0
        done = False
        epsiode_count = 0
        # 再赋予一个新的初始状态
        s = env.reset(0)
        # 设置一个死循环，后面若跳出便在死循环中跳出
        while True:
            # 每执行一个回合，count次数加1
            epsiode_count += 1
            puruser_a, puruser_a_logprob = pursuer_agent.choose_action(s)
            evader_a, evader_a_logprob = evader_agent.choose_action(s)
            # 根据参数的不同选择输出是高斯分布/Beta分布调整
            if args.policy_dist == "Beta":
                puruser_action = 2 * (puruser_a - 0.5) * args.max_action
                evader_action = 2 * (evader_a - 0.5) * args.max_action
            else:
                puruser_action = puruser_a
                evader_action = evader_a
            # 下面是执行环境交互操作
            s_, r, done = env.step(puruser_action, evader_action, epsiode_count, )  ## !!! 这里的环境是自己搭建的，输出每个人都不一样
            epsiode_reward += r

            # 下面考虑回合的最大运行次数(只要回合结束或者超过最大回合运行次数)
            if done or epsiode_count >= args.max_episode_steps:
                dw = True
            else:
                dw = False
            # 将经验存入replayBuffer中
            replay_buffer.store(s, puruser_action, puruser_a_logprob, r, s_, dw, done)
            # 重新赋值状态
            s = s_
            # 当replaybuffer尺寸到达batchsize便会开始训练
            if replay_buffer.count == args.batch_size:
                pursuer_agent.update(replay_buffer, epsiode)
                replay_buffer.count = 0
            # 如果回合结束便退出
            if done:
                epsiode_rewards.append(epsiode_reward)
                epsiode_mean_rewards.append(np.mean(epsiode_rewards))
                pbar.set_postfix({'回合': f'{epsiode}', '奖励': f'{epsiode_reward:.1f}', '平均奖励': f'{epsiode_mean_rewards[-1]:.1f}'})
                break

    # 存储训练模型
    pursuer_agent.save_checkpoint()
    # 如果需要画图的话
    if show_picture:
        pf.plot_train_reward(epsiode_rewards, epsiode_mean_rewards)

    return pursuer_agent

# 下面将用训练好的网络跑一次例程
def test_network(args, env, show_pictures=True, d_capture=0):
    epsiode_reward = 0.0
    done = False
    epsiode_count = 0
    pursuer_position = []
    escaper_position = []
    pursuer_velocity = []
    escaper_velocity = []

    env.d_capture = d_capture
    args.state_dim = env.observation_space.shape[0]
    args.action_dim = env.action_space.shape[0]
    args.max_action = float(env.action_space[0][1])
    # 下面将定义PPO智能体类
    pursuer_agent = PPO_continuous(args, 'pursuer')
    evader_agent = PPO_continuous(args, 'evader')
    pursuer_agent.load_checkpoint()

    s = env.reset(0)
    while True:
        epsiode_count += 1
        puruser_a, puruser_a_logprob = pursuer_agent.choose_action(s)
        evader_a, evader_a_logprob = evader_agent.choose_action(s)
        if args.policy_dist == "Beta":
            puruser_action = 2 * (puruser_a - 0.5) * args.max_action
            evader_action = 2 * (evader_a - 0.5) * args.max_action
        else:
            puruser_action = puruser_a
            evader_action = evader_a
        # 下面是执行环境交互操作
        s_, r, done = env.step(puruser_action, evader_action, epsiode_count, )  ## !!! 这里的环境是自己搭建的，输出每个人都不一样
        epsiode_reward += r
        pursuer_position.append(s_[6:9])
        pursuer_velocity.append(s_[9:12])
        escaper_position.append(s_[12:15])
        escaper_velocity.append(s_[15:18])

        # 下面考虑回合的最大运行次数(只要回合结束或者超过最大回合运行次数)
        if done or epsiode_count >= args.max_episode_steps:
            dw = True
        else:
            dw = False

        s = s_
        if done :
            print("当前测试得分为{}".format(epsiode_reward))
            break
    # 下面开始画图
    if show_pictures:
        pf.plot_trajectory(pursuer_position, escaper_position)

if __name__ == "__main__":
    # 声明环境
    args = args_param(max_episode_steps=64, batch_size=64, max_train_steps=40000, K_epochs=3,
                      chkpt_dir="/mnt/datab/home/yuanwenzheng/model_file/one_layer")
    # 声明参数
    env = satellites(Pursuer_position=np.array([2000000, 2000000 ,1000000]),
                     Pursuer_vector=np.array([1710, 1140, 1300]),
                     Escaper_position=np.array([1850000, 2000000, 1000000]),
                     Escaper_vector=np.array([1710, 1140, 1300]),
                     d_capture=50000,
                     args=args)
    train = False
    if train:
        pursuer_agent = train_network(args, env, show_picture=True, pre_train=True, d_capture=15000)
    else:
        test_network(args, env, show_pictures=False, d_capture=20000)
```



其中，追捕者的训练函数如下：

```cpp
def train_network(args, env, show_picture=True, pre_train=False, d_capture=0):
    epsiode_rewards = []
    epsiode_mean_rewards = []
    # 下面将导入env环境参数
    env.d_capture = d_capture
    args.state_dim = env.observation_space.shape[0]
    args.action_dim = env.action_space.shape[0]
    args.max_action = float(env.action_space[0][1])
    # 下面将定义一个缓冲区
    replay_buffer = ReplayBuffer(args)
    # 下面将定义PPO智能体类
    pursuer_agent = PPO_continuous(args, 'pursuer')
    evader_agent = PPO_continuous(args, 'evader')
    if pre_train:
        pursuer_agent.load_checkpoint()
    # 下面开始进行训练过程
    pbar = tqdm(range(args.max_train_steps), desc="Training of pursuer", unit="episode")
    for epsiode in pbar:
        # 每个回合首先对值进行初始化
        epsiode_reward = 0.0
        done = False
        epsiode_count = 0
        # 再赋予一个新的初始状态
        s = env.reset(0)
        # 设置一个死循环，后面若跳出便在死循环中跳出
        while True:
            # 每执行一个回合，count次数加1
            epsiode_count += 1
            puruser_a, puruser_a_logprob = pursuer_agent.choose_action(s)
            evader_a, evader_a_logprob = evader_agent.choose_action(s)
            # 根据参数的不同选择输出是高斯分布/Beta分布调整
            if args.policy_dist == "Beta":
                puruser_action = 2 * (puruser_a - 0.5) * args.max_action
                evader_action = 2 * (evader_a - 0.5) * args.max_action
            else:
                puruser_action = puruser_a
                evader_action = evader_a
            # 下面是执行环境交互操作
            s_, r, done = env.step(puruser_action, evader_action, epsiode_count, )  ## !!! 这里的环境是自己搭建的，输出每个人都不一样
            epsiode_reward += r

            # 下面考虑回合的最大运行次数(只要回合结束或者超过最大回合运行次数)
            if done or epsiode_count >= args.max_episode_steps:
                dw = True
            else:
                dw = False
            # 将经验存入replayBuffer中
            replay_buffer.store(s, puruser_action, puruser_a_logprob, r, s_, dw, done)
            # 重新赋值状态
            s = s_
            # 当replaybuffer尺寸到达batchsize便会开始训练
            if replay_buffer.count == args.batch_size:
                pursuer_agent.update(replay_buffer, epsiode)
                replay_buffer.count = 0
            # 如果回合结束便退出
            if done:
                epsiode_rewards.append(epsiode_reward)
                epsiode_mean_rewards.append(np.mean(epsiode_rewards))
                pbar.set_postfix({'回合': f'{epsiode}', '奖励': f'{epsiode_reward:.1f}', '平均奖励': f'{epsiode_mean_rewards[-1]:.1f}'})
                break

    # 存储训练模型
    pursuer_agent.save_checkpoint()
    # 如果需要画图的话
    if show_picture:
        pf.plot_train_reward(epsiode_rewards, epsiode_mean_rewards)

    return pursuer_agent
```

首先，初始化环境参数，包括网络输出尺度**action_dim**，输出最大值**max_action**，智能体状态尺度**state_dim**，抓捕距离**d_capture。**

定义经验池**replay_buffer**，用于存储智能体在环境中的训练数据，比如奖励，状态等。智能体需要通过经验池提取数据进行训练。

定义智能体**pursuer_agent、evader_agent**，其实也就是ppo网络的实例，用于选择动作action

开始训练：

- 环境初始化**env.reset(0)**，0表示追捕者训练，1表示逃逸者训练，2表示测试，获取初始状态s
- 智能体选择动作：pursuer_agent.choose_action(s)，从环境中获取到状态后，根据当前状态获取下一步的执行动作action
- 将智能体的动作输入值环境中，执行step函数，动作的奖励值**r**、状态**s_**、以及完成标志**done**
- 将s, puruser_action, puruser_a_logprob, r, s_, dw, done存储至经验池，保留数据以待训练
- s = s_，重新赋值状态，用于智能体选择动作
- 循环执行第二步~第五步
- 如果经验池储蓄满，更新追捕者网络参数：pursuer_agent.update(replay_buffer, epsiode)

最后，训练完成后保存训练模型

```cpp
pursuer_agent.save_checkpoint()
```

同理，逃逸星和测试函数的步骤类似，只不过测试函数不需要更新网络

### 2）environment

该文件主要负责与智能体进行交互，获取奖励值以及动作的状态值

```cpp
import numpy as np
import matplotlib.pyplot as plt
from gym import spaces
from typing import List, Optional, Dict, Tuple
import random
import scipy.stats as stats
import numpy as np
from gym import spaces
import satellite_function as sf
import gym

# 下面类用来定义环境
class satellites:

    # __annotations__用于存储变量注释信息的字典
    __annotations__ = {
        "Pursuer_position": np.ndarray,
        "Pursuer_vector": np.ndarray,
        "Escaper_position": np.ndarray,
        "Escaper_vector": np.ndarray
    }

    """
    卫星博弈环境
    

    Arguments:
        sizes (Tuple[Tuple[int]]):
        aspect_ratios (Tuple[Tuple[float]]):
    """
    def __init__(self, Pursuer_position=np.array([2000, 2000, 1000]), Pursuer_vector=np.array([1.71, 1.14, 1.3]),
                 Escaper_position=np.array([1000, 2000, 0]), Escaper_vector=np.array([1.71, 1.14, 1.3]),
                 M=0.4, dis_safe=1000, d_capture=100000, Flag=0, fuel_c=320, fuel_t=320, args=None):

        self.Pursuer_position = Pursuer_position
        self.Pursuer_vector = Pursuer_vector
        self.Escaper_position = Escaper_position
        self.Escaper_vector = Escaper_vector
        self.dis_dafe = dis_safe            # 碰撞距离
        self.d_capture = d_capture          # 抓捕距离
        self.Flag = Flag                    # 训练追捕航天器(0)、逃逸航天器(1)或者测试的标志(2)
        self.M = M                          # 航天器质量
        self.d_capture = d_capture
        self.burn_reward = 0
        self.win_reward = 100
        self.dangerous_zone = 0             # 危险区数量
        self.fuel_c = fuel_c                # 抓捕航天器燃料情况
        self.fuel_t = fuel_t  # 抓捕航天器燃料情况
        self.max_episode_steps = args.max_episode_steps
        # 下面声明其基本属性
        # position_low = np.array([-10000000, -10000000, -10000000])
        # position_high = np.array([10000000, 10000000, 10000000])
        # velocity_low = np.array([-50000, -50000, -50000])
        # velocity_high = np.array([50000, 50000, 50000])
        position_low = np.array([-500000, -500000, -500000, -10000000, -10000000, -10000000,-10000000, -10000000, -10000000])
        position_high = np.array([500000, 500000, 500000, 10000000, 10000000, 10000000, 10000000, 10000000, 10000000])
        velocity_low = np.array([-10000, -10000, -10000, -50000, -50000, -50000, -50000, -50000, -50000])
        velocity_high = np.array([10000, 10000, 10000, 50000, 50000, 50000, 50000, 50000, 50000])
        observation_low = np.concatenate((position_low, velocity_low))
        observation_high = np.concatenate((position_high, velocity_high))
        self.observation_space = spaces.Box(low=observation_low, high=observation_high, shape=(18,), dtype=np.float32)
        self.action_space = np.array([[-1.6, 1.6],
                                      [-1.6, 1.6],
                                      [-1.6, 1.6]])
        n_actions = 5  # 策略的数量
        self.action_space_beta = gym.spaces.Discrete(n_actions)    # {0,1,2,3,4}

    # 下面对Satellite的初始位置进行更新
    def reset(self, Flag):
        self.Pursuer_position = np.array([200000, 0 ,0])
        self.Pursuer_vector = np.array([0, 0, 0])
        self.pursuer_reward = 0.0
        self.Escaper_position = np.array([18000, 0, 0])
        self.Escaper_vector = np.array([0, 0, 0])
        self.escaper_reward = 0.0

        self.Flag = Flag

        s = np.array([self.Pursuer_position - self.Escaper_position, self.Pursuer_vector - self.Escaper_vector,
                      self.Pursuer_position, self.Pursuer_vector, self.Escaper_position, self.Escaper_vector]).ravel()

        return s

    def step(self, pursuer_action, escaper_action, epsiode_count):
        self.pursuer_reward = 0

        pursuer_action = [np.clip(action,-1.6,1.6) for action in pursuer_action]
        escaper_action = [np.clip(action,-1.6,1.6) for action in escaper_action]
        self.fuel_c = self.fuel_c - (np.abs(pursuer_action[0]) + np.abs(pursuer_action[1]) + np.abs(pursuer_action[2]))
        self.fuel_t = self.fuel_t - (np.abs(escaper_action[0]) + np.abs(escaper_action[1]) + np.abs(escaper_action[2]))

        dis = np.linalg.norm(self.Pursuer_position - self.Escaper_position)

        for i in range(3):      #update vector
            self.Pursuer_vector[i] += pursuer_action[i]
            self.Escaper_vector[i] += escaper_action[i]

        # 拉格朗日系数法
        # s_1, s_2 = sf.Time_window_of_danger_zone(
        #     R0_c=self.Pursuer_position,
        #     V0_c=self.Pursuer_vector,
        #     R0_t=self.Escaper_position,
        #     V0_t=self.Escaper_vector).calculation_spacecraft_status(600)

        # CW状态转移矩阵
        s_1, s_2 = sf.Clohessy_Wiltshire(
            R0_c=self.Pursuer_position,
            V0_c=self.Pursuer_vector,
            R0_t=self.Escaper_position,
            V0_t=self.Escaper_vector).State_transition_matrix(100)

        # 数值外推法
        # s_1, s_2 = sf.Numerical_calculation_method(
        #     R0_c=self.Pursuer_position,
        #     V0_c=self.Pursuer_vector,
        #     R0_t=self.Escaper_position,
        #     V0_t=self.Escaper_vector).numerical_calculation(600)

        self.Pursuer_position, self.Pursuer_vector = s_1[0:3], s_1[3:]
        self.Escaper_position, self.Escaper_vector = s_2[0:3], s_2[3:]
        self.dis = np.linalg.norm(self.Pursuer_position - self.Escaper_position)
        # print(self.dis)
        # print(np.linalg.norm(self.Pursuer_position), np.linalg.norm(self.Escaper_position))
        # print(self.dis, self.Pursuer_position, self.Escaper_position)
        # print(pursuer_action, escaper_action)

        # 求危险区数量
        self.calculate_number_hanger_area()
        print(self.dangerous_zone)

        if self.dis <= self.d_capture:
            return np.array([self.Pursuer_position - self.Escaper_position, self.Pursuer_vector - self.Escaper_vector,
                      self.Pursuer_position, self.Pursuer_vector, self.Escaper_position, self.Escaper_vector]).ravel(), \
                   self.win_reward, True
        # TODO 博弈结束判断条件后期改为燃料消耗情况，追捕成功或者燃料消耗殆尽
        elif epsiode_count >= self.max_episode_steps:
            return np.array([self.Pursuer_position - self.Escaper_position, self.Pursuer_vector - self.Escaper_vector,
                      self.Pursuer_position, self.Pursuer_vector, self.Escaper_position, self.Escaper_vector]).ravel(), \
                   self.burn_reward, True

        self.pursuer_reward = 1 if self.dis < dis else -1   # 接近目标给予奖励
        self.pursuer_reward += -1 if self.d_capture <= self.dis <= 4 * self.d_capture else -2  # 在目标距离范围内给予奖励
        self.pursuer_reward += -1 if self.dangerous_zone == 0 else self.dangerous_zone * 0.5

        pv1 = self.reward_of_action3(self.Pursuer_position)
        pv2 = self.reward_of_action1(self.Pursuer_vector)
        pv3 = self.reward_of_action2(self.Pursuer_position, self.Pursuer_vector)
        pv4 = self.reward_of_action4()
        # TODO PSO寻优或者加入指数项，随着距离的缩短每个方法所占的比例应该有所调整
        self.pursuer_reward += 1 * pv1
        self.pursuer_reward += 0.6 * pv2
        self.pursuer_reward += 0.2 * pv3
        self.pursuer_reward += 2 * pv4



        return np.array([self.Pursuer_position - self.Escaper_position, self.Pursuer_vector - self.Escaper_vector,
                      self.Pursuer_position, self.Pursuer_vector, self.Escaper_position, self.Escaper_vector]).ravel(), \
                   self.pursuer_reward, False
```

- satellites

  ：环境定义类 	

  - Pursuer_position，Pursuer_vector，Escaper_position，Escaper_vector：智能体位置速度，用xyz坐标表示
  - d_capture：抓捕距离，当追捕者和逃逸者的相对距离逼近该距离时，判定追捕者成功
  - fuel_c，fuel_t：追逃双方的燃料
  - win_reward：成功奖励值，追捕者抓捕成功后，获得该奖励值
  - observation_space：**观测空间状态，一共18位，包括追逃双方状态差值、追逃双方的状态**
  - action_space：智能体动作脉冲，上下限分别位-1.6、1.6，一共三位，包含在xyz轴
  - 其他参数：其他self参数可以忽略，在简单的追逃任务中使用不到

```cpp
    def reset(self, Flag):
        self.Pursuer_position = np.array([200000, 0 ,0])
        self.Pursuer_vector = np.array([0, 0, 0])
        self.pursuer_reward = 0.0
        self.Escaper_position = np.array([18000, 0, 0])
        self.Escaper_vector = np.array([0, 0, 0])
        self.escaper_reward = 0.0

        self.Flag = Flag

        s = np.array([self.Pursuer_position - self.Escaper_position, self.Pursuer_vector - self.Escaper_vector,
                      self.Pursuer_position, self.Pursuer_vector, self.Escaper_position, self.Escaper_vector]).ravel()

        return s
```

初始化函数，智能体获得初始状态（位置、速度以及初始奖励值），并将状态返回

```cpp
def step(self, pursuer_action, escaper_action, epsiode_count):
        self.pursuer_reward = 0

        pursuer_action = [np.clip(action,-1.6,1.6) for action in pursuer_action]
        escaper_action = [np.clip(action,-1.6,1.6) for action in escaper_action]
        self.fuel_c = self.fuel_c - (np.abs(pursuer_action[0]) + np.abs(pursuer_action[1]) + np.abs(pursuer_action[2]))
        self.fuel_t = self.fuel_t - (np.abs(escaper_action[0]) + np.abs(escaper_action[1]) + np.abs(escaper_action[2]))

        dis = np.linalg.norm(self.Pursuer_position - self.Escaper_position)

        for i in range(3):      #update vector
            self.Pursuer_vector[i] += pursuer_action[i]
            self.Escaper_vector[i] += escaper_action[i]

        # 拉格朗日系数法
        # s_1, s_2 = sf.Time_window_of_danger_zone(
        #     R0_c=self.Pursuer_position,
        #     V0_c=self.Pursuer_vector,
        #     R0_t=self.Escaper_position,
        #     V0_t=self.Escaper_vector).calculation_spacecraft_status(600)

        # CW状态转移矩阵
        s_1, s_2 = sf.Clohessy_Wiltshire(
            R0_c=self.Pursuer_position,
            V0_c=self.Pursuer_vector,
            R0_t=self.Escaper_position,
            V0_t=self.Escaper_vector).State_transition_matrix(100)

        # 数值外推法
        # s_1, s_2 = sf.Numerical_calculation_method(
        #     R0_c=self.Pursuer_position,
        #     V0_c=self.Pursuer_vector,
        #     R0_t=self.Escaper_position,
        #     V0_t=self.Escaper_vector).numerical_calculation(600)

        self.Pursuer_position, self.Pursuer_vector = s_1[0:3], s_1[3:]
        self.Escaper_position, self.Escaper_vector = s_2[0:3], s_2[3:]
        self.dis = np.linalg.norm(self.Pursuer_position - self.Escaper_position)
        # print(self.dis)
        # print(np.linalg.norm(self.Pursuer_position), np.linalg.norm(self.Escaper_position))
        # print(self.dis, self.Pursuer_position, self.Escaper_position)
        # print(pursuer_action, escaper_action)

        # 求危险区数量
        self.calculate_number_hanger_area()
        print(self.dangerous_zone)

        if self.dis <= self.d_capture:
            return np.array([self.Pursuer_position - self.Escaper_position, self.Pursuer_vector - self.Escaper_vector,
                      self.Pursuer_position, self.Pursuer_vector, self.Escaper_position, self.Escaper_vector]).ravel(), \
                   self.win_reward, True
        # TODO 博弈结束判断条件后期改为燃料消耗情况，追捕成功或者燃料消耗殆尽
        elif epsiode_count >= self.max_episode_steps:
            return np.array([self.Pursuer_position - self.Escaper_position, self.Pursuer_vector - self.Escaper_vector,
                      self.Pursuer_position, self.Pursuer_vector, self.Escaper_position, self.Escaper_vector]).ravel(), \
                   self.burn_reward, True

        self.pursuer_reward = 1 if self.dis < dis else -1   # 接近目标给予奖励
        self.pursuer_reward += -1 if self.d_capture <= self.dis <= 4 * self.d_capture else -2  # 在目标距离范围内给予奖励
        self.pursuer_reward += -1 if self.dangerous_zone == 0 else self.dangerous_zone * 0.5

        pv1 = self.reward_of_action3(self.Pursuer_position)
        pv2 = self.reward_of_action1(self.Pursuer_vector)
        pv3 = self.reward_of_action2(self.Pursuer_position, self.Pursuer_vector)
        pv4 = self.reward_of_action4()
        # TODO PSO寻优或者加入指数项，随着距离的缩短每个方法所占的比例应该有所调整
        self.pursuer_reward += 1 * pv1
        self.pursuer_reward += 0.6 * pv2
        self.pursuer_reward += 0.2 * pv3
        self.pursuer_reward += 2 * pv4



        return np.array([self.Pursuer_position - self.Escaper_position, self.Pursuer_vector - self.Escaper_vector,
                      self.Pursuer_position, self.Pursuer_vector, self.Escaper_position, self.Escaper_vector]).ravel(), \
                   self.pursuer_reward, False
```

step函数，负责智能体与环境的交互。

首先，将智能体的动作进行限制，动作上下限位[-1.6,1.6]，并计算剩余燃料（简单的减去脉冲绝对值）。

更改航天器速度，智能体的动作为脉冲，脉冲施加在航天器更改其状态。

航天器轨道外推模型：

- 拉格朗日外推模型
- CW状态转移矩阵
- 数值外推

代码分别如下：

***拉格朗日外推模型\***

```cpp
// 拉格朗日外推模型
def calculation_spacecraft_status(self, t):
        """
        :param t: 航天器轨道状态信息外推时间
        :return: 航天器轨道状态信息
        """
        def delta_E_equation_t(delta_E):
            # 根据当前时刻的平近点角差确定偏近点角差()
            eq = delta_E + sigma_t * (1 - np.cos(delta_E)) / np.sqrt(self.a_t) - \
                 (1 - self.r_t / self.a_t) * np.sin(delta_E) - np.sqrt(self.u / self.a_t**3) * t
            return eq

        def delta_E_equation_c(delta_E):
            # 根据当前时刻的平近点角差确定偏近点角差()
            eq = delta_E + sigma_c * (1 - np.cos(delta_E)) / np.sqrt(self.a_c) - \
                 (1 - self.r_c / self.a_c) * np.sin(delta_E) - np.sqrt(self.u / self.a_c**3) * t
            return eq

        def Lagrange_factor_c(sigma, delta_E):
            # 拉格朗日系数,外推轨道位置速度
            r = self.a_c + (self.r_c - self.a_c) * np.cos(delta_E) + sigma * np.sqrt(self.a_c) * np.sin(delta_E)
            F = 1 - self.a_c * (1 - np.cos(delta_E)) / self.r_c
            G = self.a_c * sigma * (1 - np.cos(delta_E)) / np.sqrt(self.u) + \
                self.r_c * np.sqrt(self.a_c / self.u) * np.sin(delta_E)
            F_c = -np.sqrt(self.u * self.a_c) * np.sin(delta_E) / (r * self.r_c)
            G_c = 1 - self.a_c * (1 - np.cos(delta_E)) / r

            return F, G, F_c, G_c
        # 偏近点角差

        sigma_t = np.dot(self.R0_t, self.V0_t) / np.sqrt(self.u)
        sigma_c = np.dot(self.R0_c, self.V0_c) / np.sqrt(self.u)
        # 当前时刻处的偏近点角差
        delta_E_t = fsolve(delta_E_equation_t, 0)
        delta_E_c = fsolve(delta_E_equation_c, 0)
        # print(self.R0_t, self.V0_t, self.R0_c, self.V0_c)
        # print(delta_E_t, sigma_c, self.a_c, self.r_c / self.a_c, np.sqrt(self.u / self.a_c ** 3) * t)
        #
        # 航天器位置速度外推(惯性系下)
        F, G, F_t, G_t = self.Lagrange_factor(sigma_t, delta_E_t)
        R_i_t = F * self.R0_t + G * self.V0_t
        V_i_t = F_t * self.R0_t + G_t * self.V0_t

        F, G, F_c, G_c = Lagrange_factor_c(sigma_c, delta_E_c)
        R_i_c = F * self.R0_c + G * self.V0_c
        V_i_c = F_c * self.R0_c + G_t * self.V0_c


        # 惯性系至近心点坐标系的转移矩阵
        C_Jg = self.coordinate_J_to_g()
        # 近心点坐标系下航天器位置
        R_i_t = np.dot(np.linalg.inv(C_Jg), R_i_t)
        V_i_t = np.dot(np.linalg.inv(C_Jg), V_i_t)
        R_i_c = np.dot(np.linalg.inv(C_Jg), R_i_c)
        V_i_c = np.dot(np.linalg.inv(C_Jg), V_i_c)

        return np.array([R_i_c, V_i_c]).ravel(), np.array([R_i_t, V_i_t]).ravel()
```

***CW状态矩阵\***

```cpp
def State_transition_matrix(self, t):
        self.t = t
        # GEO轨道半径
        R = 42164
        # target轨道半径
        # r = R + np.sqrt(self.R0_t[0]**2 + self.R0_t[0]**2 + self.R0_t[0]**2)
        r = R
        # target轨道平均角速度
        omega = math.sqrt(self.u / (r ** 3))    # 轨道平均角速度

        tau = omega * self.t
        s = np.sin(tau)
        c = np.cos(tau)
        elements = [
            [4 - 3*c, 0, 0, s/omega , 2*(1 - c)/omega, 0],
            [6*(s - tau), 1, 0, -2*(1 - c)/omega, 4*s/omega - 3*tau, 0],
            [0, 0, c, 0 , 0, s/omega],
            [3*omega*s, 0 ,0, c, 2*s, 0],
            [6*omega*(c - 1), 0, 0, -2*s, 4*c - 3, 0],
            [0, 0 , -omega*s, 0, 0, c]
        ]

        matrix = np.array(elements)
        state_c = np.array([self.R0_c,self.V0_c]).ravel()
        state_t = np.array([self.R0_t, self.V0_t]).ravel()
        state_c_new = np.dot(matrix, state_c)
        state_t_new = np.dot(matrix, state_t)

        return state_c_new, state_t_new
```



***数值外推法\***

```cpp
class Numerical_calculation_method():
    def __init__(self, R0_c=None, V0_c=None, R0_t=None, V0_t=None):
        self.R0_c = R0_c
        self.V0_c = V0_c
        self.R0_t = R0_t
        self.V0_t = V0_t
        self.u = 3.986e5
        self.pursuer_initial_state = np.concatenate((self.R0_c, self.V0_c))
        self.escaper_initial_state = np.concatenate((self.R0_t, self.V0_t))

    @staticmethod
    def orbit_ode(t, X, Tmax, direction):
        Radius = 6378
        mu = 398600
        J2 = 0
        m = 300
        r = 35786
        # r = np.sqrt((X[0]**2 + X[1]**2 + X[2]**2))
        omega = math.sqrt(mu / (r ** 3))  # 轨道平均角速度
        I = np.array([1, 0, 0])
        J = np.array([0, 1, 0])
        K = np.array([0, 0, 1])

        # 生成1*3向量，作为J2摄动的3个分量，X(1:3)为位置，X(4:6)为速度
        pJ2 = ((3 * J2 * mu * Radius ** 2) / (2 * r ** 4)) * (((X[0] / r) * (5 * (X[2] / r) ** 2 - 1)) * I +
                                                              ((X[1] / r) * (5 * (X[2] / r) ** 2 - 1)) * J +
                                                              ((X[2] / r) * (5 * (X[2] / r) ** 2 - 3)) * K)

        # 推力分量
        a_Tr = Tmax * math.cos(direction[0]) * math.cos(direction[1]) / m
        a_Tn = Tmax * math.cos(direction[0]) * math.sin(direction[1]) / m
        a_Tt = Tmax * math.sin(direction[0]) / m

        # C-W动力学方程，返回状态的导数
        dydt = [X[3], X[4], X[5],
                (2 * omega * X[4] + 3 * omega ** 2 * X[0] + a_Tr + pJ2[0]),
                (-2 * omega * X[3] + a_Tn + pJ2[1]),
                (-omega ** 2 * X[2] + a_Tt + pJ2[2])]
        return dydt

    def numerical_calculation(self, t):
        Tmax = 0
        direction = [1, 1]
        extra_params = (Tmax, direction)
        time_step = 50
        t_eval = np.arange(0, t + time_step, time_step)
        solution1 = solve_ivp(self.orbit_ode, (0, t), self.pursuer_initial_state, args=extra_params, method='RK45', t_eval=t_eval)
        solution2 = solve_ivp(self.orbit_ode, (0, t), self.escaper_initial_state, args=extra_params, method='RK45', t_eval=t_eval)
        self.R0_c = solution1.y[:3, -1]
        self.V0_c = solution1.y[3:, -1]
        self.R0_t = solution2.y[:3, -1]
        self.V0_t = solution2.y[3:, -1]

        state_c = np.array([self.R0_c, self.V0_c]).ravel()
        state_t = np.array([self.R0_t, self.V0_t]).ravel()

        return state_c, state_t
```

通过这三种外推模型进行下一状态的求解，并根据航天器新的状态求解奖励值：

具体分为三类：

```cpp
self.pursuer_reward = 1 if self.dis < dis else -1   # 接近目标给予奖励
```

如果追捕者逼近，那么奖励1

```cpp
self.pursuer_reward += -1 if self.d_capture <= self.dis <= 4 * self.d_capture else -2  # 在目标距离范围内给予奖励
```

如果追捕者保持在一定范围内，鼓励1

```cpp
        pv1 = self.reward_of_action3(self.Pursuer_position)
        pv2 = self.reward_of_action1(self.Pursuer_vector)
        pv3 = self.reward_of_action2(self.Pursuer_position, self.Pursuer_vector)
        pv4 = self.reward_of_action4()
```

其他奖励函数，这些可以自己设定

最后返回航天器最新状态，追捕者奖励值和任务成功标志位

### 3）replaybuffer

经验池代码

```cpp
import numpy as np
import torch
class ReplayBuffer:
    def __init__(self, args):
        self.state_dim = args.state_dim
        self.action_dim = args.action_dim
        self.batch_size = args.batch_size
        self.s = np.zeros((args.batch_size, args.state_dim))
        self.a = np.zeros((args.batch_size, args.action_dim))
        self.a_logprob = np.zeros((args.batch_size, args.action_dim))
        self.r = np.zeros((args.batch_size, 1))

        self.s_ = np.zeros((args.batch_size, args.state_dim))
        self.dw = np.zeros((args.batch_size, 1))
        self.done = np.zeros((args.batch_size, 1))
        self.count = 0

    def store(self, s, a, a_logprob,r, s_, dw, done):
        index = self.count % self.batch_size  # Calculate the index considering batch size
        self.s[index] = s
        self.a[index] = a
        self.a_logprob[index] = a_logprob
        self.r[index] = r
        self.s_[index] = s_
        self.dw[index] = dw
        self.done[index] = done
        self.count += 1


    def numpy_to_tensor(self):
        s = torch.tensor(self.s, dtype=torch.float)
        a = torch.tensor(self.a, dtype=torch.float)
        a_logprob = torch.tensor(self.a_logprob, dtype=torch.float)
        r = torch.tensor(self.r, dtype=torch.float)
        s_ = torch.tensor(self.s_, dtype=torch.float)
        dw = torch.tensor(self.dw, dtype=torch.float)
        done = torch.tensor(self.done, dtype=torch.float)
        return s, a, a_logprob, r, s_, dw, done
```

### 4）ppo

PPO网络文件，包括ac框架以及网络更新机制

```cpp
# --coding:utf-8--
import torch
import torch.nn.functional as F
from torch.utils.data.sampler import BatchSampler, SubsetRandomSampler
import torch.nn as nn
from torch.distributions import Beta, Normal
import os

# Trick 8: orthogonal initialization
def orthogonal_init(layer, gain=1.0):
    nn.init.orthogonal_(layer.weight, gain=gain)
    nn.init.constant_(layer.bias, 0)

class Actor_Beta(nn.Module):
    def __init__(self, args, agent_idx):
        super(Actor_Beta, self).__init__()
        chkpt_dir = args.chkpt_dir
        self.agent_name = 'agent_%s' % agent_idx
        self.chkpt_file = os.path.join(chkpt_dir, self.agent_name + '_actor_Beta')

        self.fc1 = nn.Linear(args.state_dim, args.hidden_width)
        self.fc2 = nn.Linear(args.hidden_width, args.hidden_width)
        # self.fc3 = nn.Linear(args.hidden_width2, args.hidden_width2)
        self.alpha_layer = nn.Linear(args.hidden_width, args.action_dim)
        self.beta_layer = nn.Linear(args.hidden_width, args.action_dim)
        self.activate_func = [nn.ReLU(), nn.Tanh()][args.use_tanh]  # Trick10: use tanh

        if args.use_orthogonal_init:
            print("------use_orthogonal_init------")
            orthogonal_init(self.fc1)
            orthogonal_init(self.fc2)
            # orthogonal_init(self.fc3)
            orthogonal_init(self.alpha_layer, gain=0.01)
            orthogonal_init(self.beta_layer, gain=0.01)

    def forward(self, s):
        s = self.activate_func(self.fc1(s))
        s = self.activate_func(self.fc2(s))
        # s = self.activate_func(self.fc3(s))
        # alpha and beta need to be larger than 1,so we use 'softplus' as the activation function and then plus 1
        alpha = F.softplus(self.alpha_layer(s)) + 1.0
        beta = F.softplus(self.beta_layer(s)) + 1.0
        return alpha, beta

    def get_dist(self, s):
        alpha, beta = self.forward(s)
        dist = Beta(alpha, beta)
        return dist

    def mean(self, s):
        alpha, beta = self.forward(s)
        mean = alpha / (alpha + beta)  # The mean of the beta distribution
        return mean

    def save_checkpoint(self):
        torch.save(self.state_dict(), self.chkpt_file)

    def load_checkpoint(self):
        self.load_state_dict(torch.load(self.chkpt_file))

class Actor_Gaussian(nn.Module):
    def __init__(self, args, agent_idx):
        super(Actor_Gaussian, self).__init__()
        chkpt_dir = args.chkpt_dir
        self.agent_name = 'agent_%s' % agent_idx
        self.chkpt_file = os.path.join(chkpt_dir, self.agent_name + '_actor_Gaussian')

        self.max_action = args.max_action
        self.fc1 = nn.Linear(args.state_dim, args.hidden_width)
        self.fc2 = nn.Linear(args.hidden_width, args.hidden_width)
        # self.fc3 = nn.Linear(args.hidden_width2, args.hidden_width2)
        self.mean_layer = nn.Linear(args.hidden_width, args.action_dim)
        self.log_std = nn.Parameter(torch.zeros(1, args.action_dim))  # We use 'nn.Parameter' to train log_std automatically
        self.activate_func = [nn.ReLU(), nn.Tanh()][args.use_tanh]  # Trick10: use tanh

        if args.use_orthogonal_init:
            print("------use_orthogonal_init------")
            orthogonal_init(self.fc1)
            orthogonal_init(self.fc2)
            # orthogonal_init(self.fc3)
            orthogonal_init(self.mean_layer, gain=0.01)

    def forward(self, s):
        s = self.activate_func(self.fc1(s))
        s = self.activate_func(self.fc2(s))
        # s = self.activate_func(self.fc3(s))
        mean = self.max_action * torch.tanh(self.mean_layer(s))  # [-1,1]->[-max_action,max_action]
        return mean

    def get_dist(self, s):
        mean = self.forward(s)
        log_std = self.log_std.expand_as(mean)  # To make 'log_std' have the same dimension as 'mean'
        std = torch.exp(log_std)  # The reason we train the 'log_std' is to ensure std=exp(log_std)>0
        dist = Normal(mean, std)  # Get the Gaussian distribution
        return dist

    def save_checkpoint(self):
        torch.save(self.state_dict(), self.chkpt_file)

    def load_checkpoint(self):
        self.load_state_dict(torch.load(self.chkpt_file))

class Critic(nn.Module):
    def __init__(self, args, agent_idx):
        super(Critic, self).__init__()
        chkpt_dir = args.chkpt_dir
        self.agent_name = 'agent_%s' % agent_idx
        self.chkpt_file = os.path.join(chkpt_dir, self.agent_name + '_critic')

        self.fc1 = nn.Linear(args.state_dim, args.hidden_width)
        self.fc2 = nn.Linear(args.hidden_width, args.hidden_width)
        # self.fc3 = nn.Linear(args.hidden_width2, args.hidden_width2)
        self.fc3 = nn.Linear(args.hidden_width, 1)
        self.activate_func = [nn.ReLU(), nn.Tanh()][args.use_tanh]  # Trick10: use tanh

        if args.use_orthogonal_init:
            print("------use_orthogonal_init------")
            orthogonal_init(self.fc1)
            orthogonal_init(self.fc2)
            orthogonal_init(self.fc3)
            # orthogonal_init(self.fc4)

    def forward(self, s):
        s = self.activate_func(self.fc1(s))
        s = self.activate_func(self.fc2(s))
        # s = self.activate_func(self.fc3(s))
        v_s = self.fc3(s)
        return v_s

    def save_checkpoint(self):
        torch.save(self.state_dict(), self.chkpt_file)

    def load_checkpoint(self):
        self.load_state_dict(torch.load(self.chkpt_file))

class PPO_continuous():
    def __init__(self, args, agent_idx):
        self.policy_dist = args.policy_dist
        self.max_action = args.max_action
        self.batch_size = args.batch_size
        self.mini_batch_size = args.mini_batch_size
        self.max_train_steps = args.max_train_steps
        self.lr_a = args.lr_a  # Learning rate of actor
        self.lr_c = args.lr_c  # Learning rate of critic
        self.gamma = args.gamma  # Discount factor
        self.lamda = args.lamda  # GAE parameter
        self.epsilon = args.epsilon  # PPO clip parameter
        self.K_epochs = args.K_epochs  # PPO parameter
        self.entropy_coef = args.entropy_coef  # Entropy coefficient
        self.set_adam_eps = args.set_adam_eps
        self.use_grad_clip = args.use_grad_clip
        self.use_lr_decay = args.use_lr_decay
        self.use_adv_norm = args.use_adv_norm

        if self.policy_dist == "Beta":
            self.actor = Actor_Beta(args, agent_idx)
        else:
            self.actor = Actor_Gaussian(args, agent_idx)
        self.critic = Critic(args, agent_idx)

        if self.set_adam_eps:  # Trick 9: set Adam epsilon=1e-5
            self.optimizer_actor = torch.optim.Adam(self.actor.parameters(), lr=self.lr_a, eps=1e-5)
            self.optimizer_critic = torch.optim.Adam(self.critic.parameters(), lr=self.lr_c, eps=1e-5)
        else:
            self.optimizer_actor = torch.optim.Adam(self.actor.parameters(), lr=self.lr_a)
            self.optimizer_critic = torch.optim.Adam(self.critic.parameters(), lr=self.lr_c)

    def evaluate(self, s):  # When evaluating the policy, we only use the mean
        s = torch.unsqueeze(torch.tensor(s, dtype=torch.float), 0)
        if self.policy_dist == "Beta":
            a = self.actor.mean(s).detach().numpy().flatten()
        else:
            a = self.actor(s).detach().numpy().flatten()
        return a

    def choose_action(self, s):
        s = torch.unsqueeze(torch.tensor(s, dtype=torch.float), 0)
        if self.policy_dist == "Beta":
            with torch.no_grad():
                dist = self.actor.get_dist(s)
                a = dist.sample()  # Sample the action according to the probability distribution
                a_logprob = dist.log_prob(a)  # The log probability density of the action
        else:
            with torch.no_grad():
                dist = self.actor.get_dist(s)
                a = dist.sample()  # Sample the action according to the probability distribution
                a = torch.clamp(a, -self.max_action, self.max_action)  # [-max,max]
                a_logprob = dist.log_prob(a)  # The log probability density of the action
        return a.numpy().flatten(), a_logprob.numpy().flatten()

    def update(self, replay_buffer, total_steps):
        s, a, a_logprob, r, s_, dw, done = replay_buffer.numpy_to_tensor()  # Get training data
        """
            Calculate the advantage using GAE
            'dw=True' means dead or win, there is no next state s'
            'done=True' represents the terminal of an episode(dead or win or reaching the max_episode_steps). When calculating the adv, if done=True, gae=0
        """
        adv = []
        gae = 0
        with torch.no_grad():  # adv and v_target have no gradient
            vs = self.critic(s)
            vs_ = self.critic(s_)
            deltas = r + self.gamma * (1.0 - dw) * vs_ - vs
            for delta, d in zip(reversed(deltas.flatten().numpy()), reversed(done.flatten().numpy())):
                gae = delta + self.gamma * self.lamda * gae * (1.0 - d)
                adv.insert(0, gae)
            adv = torch.tensor(adv, dtype=torch.float).view(-1, 1)
            v_target = adv + vs
            if self.use_adv_norm:  # Trick 1:advantage normalization
                adv = ((adv - adv.mean()) / (adv.std() + 1e-5))

        # Optimize policy for K epochs:
        for _ in range(self.K_epochs):
            # Random sampling and no repetition. 'False' indicates that training will continue even if the number of samples in the last time is less than mini_batch_size
            for index in BatchSampler(SubsetRandomSampler(range(self.batch_size)), self.mini_batch_size, False):
                dist_now = self.actor.get_dist(s[index])
                dist_entropy = dist_now.entropy().sum(1, keepdim=True)  # shape(mini_batch_size X 1)
                a_logprob_now = dist_now.log_prob(a[index])
                # a/b=exp(log(a)-log(b))  In multi-dimensional continuous action space，we need to sum up the log_prob
                ratios = torch.exp(a_logprob_now.sum(1, keepdim=True) - a_logprob[index].sum(1, keepdim=True))  # shape(mini_batch_size X 1)

                surr1 = ratios * adv[index]  # Only calculate the gradient of 'a_logprob_now' in ratios
                surr2 = torch.clamp(ratios, 1 - self.epsilon, 1 + self.epsilon) * adv[index]
                actor_loss = -torch.min(surr1, surr2) - self.entropy_coef * dist_entropy  # Trick 5: policy entropy
                # Update actor
                self.optimizer_actor.zero_grad()
                actor_loss.mean().backward()
                if self.use_grad_clip:  # Trick 7: Gradient clip
                    torch.nn.utils.clip_grad_norm_(self.actor.parameters(), 0.5)
                self.optimizer_actor.step()

                v_s = self.critic(s[index])
                critic_loss = F.mse_loss(v_target[index], v_s)
                # Update critic
                self.optimizer_critic.zero_grad()
                critic_loss.backward()
                if self.use_grad_clip:  # Trick 7: Gradient clip
                    torch.nn.utils.clip_grad_norm_(self.critic.parameters(), 0.5)
                self.optimizer_critic.step()

        if self.use_lr_decay:  # Trick 6:learning rate Decay
            self.lr_decay(total_steps)

    def lr_decay(self, total_steps):
        lr_a_now = self.lr_a * (1 - total_steps / self.max_train_steps)
        lr_c_now = self.lr_c * (1 - total_steps / self.max_train_steps)
        for p in self.optimizer_actor.param_groups:
            p['lr'] = lr_a_now
        for p in self.optimizer_critic.param_groups:
            p['lr'] = lr_c_now

    def save_checkpoint(self):
        self.actor.save_checkpoint()
        self.critic.save_checkpoint()

    def load_checkpoint(self):
        self.actor.load_checkpoint()
        self.critic.load_checkpoint()
```

### 5）训练

```cpp
if __name__ == "__main__":
    # 声明环境
    args = args_param(max_episode_steps=64, batch_size=64, max_train_steps=40000, K_epochs=3,
                      chkpt_dir="/mnt/datab/home/yuanwenzheng/model_file/one_layer")
    # 声明参数
    env = satellites(Pursuer_position=np.array([2000000, 2000000 ,1000000]),
                     Pursuer_vector=np.array([1710, 1140, 1300]),
                     Escaper_position=np.array([1850000, 2000000, 1000000]),
                     Escaper_vector=np.array([1710, 1140, 1300]),
                     d_capture=50000,
                     args=args)
    train = False
    if train:
        pursuer_agent = train_network(args, env, show_picture=True, pre_train=True, d_capture=15000)
    else:
        test_network(args, env, show_pictures=False, d_capture=20000)
```

- **chkpt_dir**：模型文件地址，如果进行测试test，那么需要把该地址设置为你的模型文件地址
- **env** ：可手动修改航天器初始状态
- **train**：训练标志，true为训练，false为测试

训练过程：

![img](/images/$%7Bfiilename%7D/e79d5c0bd3d243d881a03ae7b96df954.png)

训练结果：

![img](/images/$%7Bfiilename%7D/637223b5a2794e24b8ad6181fa601a23.png)

测试：

![img](/images/$%7Bfiilename%7D/6b3b88bb82bd4ccaad6c914481591c1d.png)

分数很高，以及成功抓捕



参考：

[基于强化学习的空战辅助决策(2D)_afsim开源代码-CSDN博客blog.csdn.net/shengzimao/article/details/126787045](https://link.zhihu.com/?target=https%3A//blog.csdn.net/shengzimao/article/details/126787045)
