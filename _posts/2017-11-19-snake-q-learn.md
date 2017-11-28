---
layout: post
title:  "Using Reinforcement-learning and Q-learning to play Snake"
date:   2017-11-19
categories: walkthrough
comments: true
---
## Dependencies
- Python3.6
- Keras
- Scikit-image
- Tensorflow

## Background
The background for me to do this project was of course to learn more about reinforced learning but also to get a trip down memory lane with the classic Snake game. I have previously done some work with other deep learning methods such as classifications tasks so it was rewarding to try something slightly different.


  ![]({{ https://hugocool24.github.io/ }}/images/snake.gif)
## Reinforcement Learning
In the Snake game we are controlling a snake that wants to eat apples(weird right?!) and when it does it gains a reward and it also grows in size. The snake is allowed to move freely around the playing field but the game is over if the snake hits the walls or itself. To control it we can use four different actions, arrow up, down, left and right.

Now how would we teach a neural network to solve this game? If we have a picture from the game and then from that picture we determine what action to take, up, down, left or right. So we have a classic classification problem ahead of us.

Now if we have a classification problem we need a-lot of data to train on, since we don't have any data and I lost interest playing Snake in 4th grade we need to automatically create data. So instead of playing the game myself I let an agent play the game and perform random actions and then the game return a reward based on the action.

It is recommended that a reward shall be somewhere between-1 and 1 therefore I chose the reward when the snake eats the apple to be 1 and -1 when the snake dies and 0 otherwise the reward is 0. Based on the reward the agent should learn how to play the game.

In many ways the way that the agent learns to play a game is similar to how animals/humans learns thing. For an example if you want to learn a dog how to sit, so tell it to sit and in a beginning it doesn't grasp what we are saying to it however after a while it will sit down and then get a reward. The next time we tell the dog to sit it will connect the dots and sit down and get a reward. As in the game of Snake the dog can also reach a local minimum when we just show the dog candy without telling it to sit it will sit down as soon as it sees the candy and still get candy without us saying sit.

An example of what happens when the game reaches a local minimum would be that the snake is going in circles around the middle of the game because it knows the apple has a higher probability of appearing there. To avoid this we have to give it some kind reward if the agent does an action that is good in the long term versus good in a short term.

## Convolutional Neural Network
A Convolutional Neural Network or CNN is a kind of network that is suitable to use when the input format to the neural net is a matrix or tensor instead of a vector. In our case the input we use is a 80x80x1 image. A convolution is a sliding filter that gets applied to the picture. In the gif below a 3x3 kernel filter it applied to a black and white picture. Where the 0 is black and 1 is white. The filter multiplies the values element-wise and sums them up.

![]({{ https://hugocool24.github.io/ }}/images/Convolution_schematic.gif)

How this works in practice is somewhat shown below where a kernel filter is applied to a picture of Taj Mahal.

![]({{ https://hugocool24.github.io/ }}/images/convolution-edge-detect1.png)
![]({{ https://hugocool24.github.io/ }}/images/generic-taj-convmatrix-edge-detect.jpg)

![]({{ https://hugocool24.github.io/ }}/images/convolution-calculate.png)

The way the filter is applied is also shown in the picture above where the filter reads from left to right, top to down. With the filter applied to the matrix the result is 42: (40\*0)+(42\*1)+(46\*0) + (46\*0)+(50\*0)+(55\*0) + (52\*0)+(56\*0)+(58\*0) = 42.

I used [this](http://www.wildml.com/2015/11/understanding-convolutional-neural-networks-for-nlp/) and [this](https://docs.gimp.org/en/plug-in-convmatrix.html) resource to help me explain what a convolutional neural network is, the article also explains more of the different kinds of layers and also how a CNN can be used for NLP(Natural Language Processing)


## Q-learn

The Q-Learning algorithm goes as follows:

1. Set the gamma parameter, and environment rewards in matrix R.

2. Initialize matrix Q to zero.

3. For each episode:

	- Select a random initial state.
	- Do While the goal state hasn't been reached.
	- Select one among all possible actions for the current state.
	- Using this possible action, go to the next state.
	- Get maximum Q value for this next state based on all possible actions.
	- Compute: Q(state, action) = R(state, action) + Gamma * Max[Q(next state, Action with highest Q-value)]
	- Set the next state as the current state.

The gamma parameter is responsible for how long into the future we want to be the reward on, is has a range between 0 and 1. The close gamma is to zero the agent will consider immediate rewards more and if gamma is close to 1 the agent will consider future rewards more and can therefore delay the reward.

The state is the current picture of the game, where the snake and the apple is in the game.
Actions are the possible actions that the agent can do.

The Q-value is the brain of the agent, it is the memory of the past actions and rewards of the agent.

## Code walkthrough

Let's walk through some parts of the code.

{% highlight python linenos %}
def build_model():
    model = Sequential()
    model.add(Convolution2D(16, (8, 8), strides=(4, 4),input_shape=INPUT_SHAPE))
    model.add(Activation('relu'))
    model.add(Convolution2D(32, (4, 4), strides=(2, 2)))
    model.add(Activation('relu'))
    model.add(Flatten())
    model.add(Dense(256))
    model.add(Dense(NB_ACTIONS))
    adam=Adam(lr=LEARNING_RATE)
    model.compile(loss='mean_squared_error',
                    optimizer=adam)
    print(model.summary())
    return model
{% endhighlight %}

In this part of the code we build up the CNN.

- The third row describes the first convolution layer, 16 means that the dimension of the output space of the convolution.
- (8, 8) is the size of the kernel filter.
- Strides refers to how the kernel moves over the images, now it moves with 4, 4 pixels. - input_shape is the dimension of the picture that we feed to the convolution layer.
- The layer that says Dense(NB_ACTIONS) is the layer that makes sure that we get 4 possible actions out from the network.

{% highlight python linenos %}
def stack_image(game_image):
    #Make image black and white
    x_t = skimage.color.rgb2gray(game_image)
    #Resize the image to 80x80 pixels
    x_t = skimage.transform.resize(x_t,(80,80))
    #Change the intensity of colors, maximizing the intensities.
    x_t = skimage.exposure.rescale_intensity(x_t,out_range=(0,255))
    # Stacking 2 images for the agent to get understanding of speed
    s_t = np.stack((x_t,x_t),axis=2)
    # Reshape to make keras like it
    s_t = s_t.reshape(1, s_t.shape[0], s_t.shape[1], s_t.shape[2])
    return s_t
{% endhighlight %}
This part of the code actually has good comments describing each step.

{% highlight python linenos%}
def train_network(model):

    game_state = game.Game() #Starting up a game
    game_state.set_start_state()
    game_image,score,game_lost = game_state.run(0) #The game is started but no action is performed
    s_t = stack_image(game_image)
    terminal = False
    t = 0
    d = []
    nb_epoch = 0
    while(True):
        loss = 0
        Q_sa = 0
        action_index = 4
        r_t = 0
        a_t = 'no nothing'
        if terminal:
            game_state.set_start_state()
        if t % NB_FRAMES == 0:
            if random.random() <= EPSILON:
                action_index = random.randrange(NB_ACTIONS)
                a_t = GAME_INPUT[action_index]
            else:
                action_index = np.argmax(model.predict(s_t))
                a_t = GAME_INPUT[action_index]
        #run the selected action and observed next state and reward
	    x_t1_colored, r_t, terminal = game_state.run(a_t)
	    s_t1 = stack_image(x_t1_colored)
	    d.append((s_t, a_t, r_t, s_t1))
{% endhighlight %}

First the game i started and gives back information about the initial state and also gets an image from the game. I initialize loss and Q_sa to 0. If the game is lost I the game is restarted.

Then the agent will make a random action or predict an action. This is to make sure that we throw in some random actions here and there so the agent doesn't get stuck in a local minimum.

Then the action is fed to the game and we receive an image with the new state and also state information such as reward and is the game is over or not.

{% highlight python linenos %}
	    if len(d)==BATCH:
	        inputs = np.zeros((BATCH, s_t.shape[1], s_t.shape[2], s_t.shape[3]))
	        targets = np.zeros((BATCH, NB_ACTIONS))
	        i = 0
	        for s,a,r,s_pred in d:
	            inputs[i:i + 1] = s
	            if r < 0:
	                targets[i ,a] = r
	            else:
	                Q_sa = model.predict(s_pred)
	                targets[i ,a] = r + GAMMA * np.max(Q_sa)
	            i+=1
	        loss += model.train_on_batch(inputs,targets)
	        d.clear()
	        #Exploration vs Exploitation
	        if EPSILON > FINAL_EPSILON:
	            EPSILON -= EPSILON/500
{% endhighlight %}

Here is the part where we do perform the Q-learning. We loop through the current state, action correlating with that state, the reward and also the next state. For each state we calculate the corresponding Q-value and then we train our model on the small batch.

Lastly we make epsilon smaller and smaller by each episode. This is so we get less and less random actions.

## Thank you

Thank you for taking you time to read through this. If you have any questions or comments feel free to drop a comment below or send me an email.

## Disclaimer

This post has been influenced by [this](https://yanpanlau.github.io/2016/07/10/FlappyBird-Keras.html) post, describing how q-learn can be used to play Flappy Bird.
