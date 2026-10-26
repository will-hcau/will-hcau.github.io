---
title: "Crack flappy bird with reinforcement learning"
date: 2021-05-04 00:00:00 +0900
categories: [Machine Learning]
tags: [reinforcement-learning, machine-learning, ai, double-q-learning, flappy-bird]
description: "My 2021 experiment training a Flappy Bird agent with deep Q-learning, a target network, experience replay, Double Q-learning, and a dueling network."
---

> Originally published on [Medium on May 4, 2021](https://medium.com/@willhcau/crack-flappy-bird-with-reinforcement-learning-9dbabea9de3f). Migrated to this blog with the original text and results preserved. Framework names and environment links reflect the original 2021 article.
{: .prompt-info }

![Flappy Bird](/assets/img/flappy-bird-rl/flappy-bird.jpeg)
_Flappy Bird_

After learning basic knowledge of deep reinforcement learning algorithm, I started to think about implementing something interesting to practice. I have already train agents to solve simple openAI gym games like CartPole, Pendulum and LunarLander. Now let’s looks for something more interesting and the first thing comes to my mind is Flappy bird. A well known game which human found it very difficult to get high score.

{% include embed/youtube.html id='HJLAqdpsNo4' %}
_RL playing flappy bird_

## Q-Learning

A reinforcement learning task is about training an **agent** which interact with **environment**. The agent fall into difference scenario knows as **state** by performing **actions**. Action will leads to **rewards** which could be positive or negative depends on the design of environment.

The agent has only one objective to maximize the accumulated reward within an **episode**. An episode is whatever happen from the 1st start util the terminal state. The algorithm reinforce the agent to learn to performance the best action under certain state, the learned strategy is known as **policy**.

![Q-Learning algorithm from the 2015 DQN paper](/assets/img/flappy-bird-rl/dqn-2015.png)
_Q-Learning Algorithm (Human-level control through deep reinforcement learning, Mnih et al., 2015)_

The 2015 version of DQN paper is originally base on the 2013 DQN paper which firstly proposed the DQN to play Atari games. And the 2015 version has two add-on compare to the pure Q-Learning algorithm. They are: Target Network and Experience replay buffer.

### Target Network

![Original algorithm from the 2013 DQN paper](/assets/img/flappy-bird-rl/dqn-2013.png)
_The original 2013 Q-Learning Algorithm (Playing Atari with Deep Reinforcement Learning, Mnih et al., 2013)_

As you can see from the original 2013 Q-Learning algorithm, every step we are optimizing the Q-function toward Y where Y is obtained from the same Q-function. The Q-function is then being update each of the step, its like we are chasing toward a moving target. The variance can be great, thus slow down the training process.

So the 2015 Q-Learning paper improved it by using a separated target network Q-hat. Target network has the same architecture as the function approximator but with frozen parameters. For every C steps (a hyperparameter), the parameter from predictor network is copy to the target network. That makes the training more stable.

### Experience Replay Buffer

The experience replay buffer is to store the agent’s transition (𝑠𝑡,𝑎𝑡,𝑟𝑡,𝑠𝑡+1)

It basically a larger buffer to hold agent’s experience at each time stamp accumulated though many episode. We then uniformly sample minibatch from the buffer to learn off-policy. Experience replay buffer breaks the correlation between steps, and allow experience to be reused multiple times.

## Other advanced Q-Learning techniques

### Double Q Learning

Double Q Learning was proposed for solving the problem of over estimation of Q value. The optimal policy choose the best action at a given state base on the estimated Q value which is produced by an estimated Q function. However the agent knows nothing about the environment at the beginning and asking it the estimate Q(s,a) will produce a lots of noise. Taking the maximum value introduce bias in learning, it’s kind of learning estimates from estimates which can be problematic.

![Double Q-learning equation](/assets/img/flappy-bird-rl/double-q-learning.png)
_Deep Reinforcement Learning with Double Q-learning (2015)_

To overcome the overestimation problem, we need two model: one to estimation the best action, another to estimate Q value. The idea is we accept the fact that Q value can be overestimated under noise. And by separating the model for selecting action and estimating Q value, we can reduce the chance of overestimation. Because if Q(s,a) determined a maximizing action which may be overestimated, hopefully the Q^hat(s,a) will have a correct estimation even through the action selected is not optimal.

I am not going into details as here is just a brief introduction. There are lots resources on the internet and also the paper is public. Please refer to external resources if you are interested.

### Dueling Network

Another advanced technique is the dueling network. A popular single stream Q-network (top) and the dueling Q-network (bottom). The dueling network has two streams to separately estimate (scalar) state-value and the advantages for each action; the green output module implements equation (9) to combine them. Both networks output Q-values for each action.

![Single-stream and dueling Q-network architectures](/assets/img/flappy-bird-rl/dueling-network.png)
_Dueling Network Architecture_

The author proposed in most of the situation, estimating action is not necessary. Like playing a racing game, you don’t need to turn unless the race car is going to crash. So instead to predict turn left or turn right, we can modify the network architecture so that it learns in what status the agent should make decision, in what status the agent can stay still. In this way the training efficiency will be increase.

## Environment

### Game environment

Thanks to Luis Sobrecueva’s FlappyBird implementation, which is compatible to OpenAI GYM environment. So I don’t need to spend time on making game from scratch and put my focus on RL algorithm.

[OpenAI Gym: FlappyBird-v0](https://gym.openai.com/envs/FlappyBird-v0/) — original environment link from 2021.

![Modified Flappy Bird game rules and rewards](/assets/img/flappy-bird-rl/game-rules.png)
_Game rule_

In fact, I modified a little bit of the original game environment. Originally there is no ceiling at the top, the bird can fly up infinity high off-screen. I found that cause the agent having trouble to get through the tube at the beginning of training. So I add ceiling boundary to limit the exploration space thus speed up training. Second, I also modified the rewarding rule as shown above to encourage the agent for passing through a tube.

### RL Framework

Instead of using those available RL framework on the community, I implement my own RL framework as a practice. Expect the fundamentals Q-Learning algorithm, I also implement a few advanced tools for example: Double Q, Dueling Network and Multi Step Learning.

[will-hcau/rlkeras](https://github.com/will-hcau/rlkeras)

## Experiment

![CNN architecture of the Flappy Bird agent](/assets/img/flappy-bird-rl/cnn-architecture.png)
_CNN architecture of the flappy bird agent_

The setup is basically the same as the 2015 DQN paper. Four consecutive frame is capture and converted into grey scale. It goes through layers of convolution layer and a few dense layer. Finally it calculate two output corresponding to two possible action: no action & jump. Also putting all advanced technique mentioned before, I try to train an agent to play flappy bird with the following setup.

- Input: Four grey scale 80 x 80 game screen concatenated
- Action output: 0 or 1 (0: no action, 1: jump)
- Double Q: Enable
- Dueling Network: Enable
- Memory replay buffer size: 50000
- Batch Size: 32
- Target Mode Update: 10000 steps
- Learning Rate (Adam): 0.00001
- Total training step: 1000000

## Result

I am training with a laptop CPU which doesn’t allow me to train fast for better result. Spending a few days, only a million step was trained. So I don’t expect the agent will perform very well at the current state. But I am pretty happy to see it can score over two hundred points on average.

### Episode results

The following log records the reward and step count for 50 episodes. **Reward is the environment's accumulated reward, not necessarily the number of pipes passed.**

```text
Episode 1: reward: 432.600, steps: 3500
Episode 2: reward: 51.100, steps: 423
Episode 3: reward: 239.600, steps: 1939
Episode 4: reward: 97.100, steps: 793
Episode 5: reward: 30.800, steps: 256
Episode 6: reward: 64.800, steps: 533
Episode 7: reward: 57.700, steps: 471
Episode 8: reward: 6.100, steps: 63
Episode 9: reward: 289.400, steps: 2347
Episode 10: reward: 74.800, steps: 615
Episode 11: reward: 177.700, steps: 1437
Episode 12: reward: 46.400, steps: 385
Episode 13: reward: 72.000, steps: 587
Episode 14: reward: 228.300, steps: 1844
Episode 15: reward: 94.600, steps: 768
Episode 16: reward: 18.800, steps: 163
Episode 17: reward: 433.700, steps: 3502
Episode 18: reward: 138.400, steps: 1125
Episode 19: reward: 102.600, steps: 839
Episode 20: reward: 21.400, steps: 180
Episode 21: reward: 210.500, steps: 1702
Episode 22: reward: 60.300, steps: 497
Episode 23: reward: 708.800, steps: 5713
Episode 24: reward: 334.000, steps: 2694
Episode 25: reward: 287.800, steps: 2322
Episode 26: reward: 368.400, steps: 2975
Episode 27: reward: 122.400, steps: 992
Episode 28: reward: 46.400, steps: 385
Episode 29: reward: 53.200, steps: 435
Episode 30: reward: 199.200, steps: 1616
Episode 31: reward: 51.900, steps: 431
Episode 32: reward: 41.900, steps: 349
Episode 33: reward: 163.900, steps: 1326
Episode 34: reward: 63.300, steps: 518
Episode 35: reward: 16.400, steps: 139
Episode 36: reward: 194.500, steps: 1578
Episode 37: reward: 35.300, steps: 292
Episode 38: reward: 46.400, steps: 385
Episode 39: reward: 25.800, steps: 215
Episode 40: reward: 37.200, steps: 311
Episode 41: reward: 79.100, steps: 649
Episode 42: reward: 368.400, steps: 2975
Episode 43: reward: 56.200, steps: 465
Episode 44: reward: 18.800, steps: 163
Episode 45: reward: 248.800, steps: 2013
Episode 46: reward: 64.800, steps: 533
Episode 47: reward: 74.000, steps: 607
Episode 48: reward: 221.200, steps: 1791
Episode 49: reward: 5.900, steps: 61
Episode 50: reward: 94.500, steps: 776
```
