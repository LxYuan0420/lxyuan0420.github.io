---
layout: post
title: "My first reinforcement learning model on AWS"
description: "This article demonstrates my experience of training a reinforcement learning (RL) model on AWS."
---

Recently, I signed up for a free-tier account on AWS to delve deeper into their
various services, and that's when I stumbled upon AWS DeepRacer—a thrilling car
racing game based on reinforcement learning. I found it interesting
and thought it was definitely worth a try, especially considering that new
users receive 10 free hours to train or evaluate models, along with 5GB of free
storage during the first month. Why not take advantage of this opportunity?

In DeepRacer, we have the opportunity to define our desired behavior and
indicate what is considered favorable or unfavorable to the car by creating a
Python function that returns a floating-point reward. This function serves as a
guiding principle, helping the car distinguish between right and wrong actions.
Additionally, we can configure training scenarios by selecting virtual tracks,
adjusting training durations, and fine-tuning the car's steering and
acceleration sensitivity. During training, progress is constantly monitored,
measuring the percentage of the track completed before veering off-course. To
further assess the car's performance, we can evaluate and monitor its behavior
through virtual evaluation videos after each training session.

Below is the python-based reward function I used:
{% highlight python %}
import math
def reward_function(params):

    # Reward weights
    speed_weight = 100
    heading_weight = 100
    steering_weight = 50

    # Initialize the reward based on current speed
    max_speed_reward = 10 * 10
    min_speed_reward = 3.33 * 3.33
    abs_speed_reward = params['speed'] * params['speed']
    speed_reward = (abs_speed_reward - min_speed_reward) / (max_speed_reward - min_speed_reward) * speed_weight
    
    # - - - - - 
    
    # Penalize if the car goes off track
    if not params['all_wheels_on_track']:
        return 1e-3
    
    # - - - - - 
    
    # Calculate the direction of the center line based on the closest waypoints
    next_point = params['waypoints'][params['closest_waypoints'][1]]
    prev_point = params['waypoints'][params['closest_waypoints'][0]]

    # Calculate the direction in radius, arctan2(dy, dx), the result is (-pi, pi) in radians
    track_direction = math.atan2(next_point[1] - prev_point[1], next_point[0] - prev_point[0]) 
    # Convert to degree
    track_direction = math.degrees(track_direction)

    # Calculate the difference between the track direction and the heading direction of the car
    direction_diff = abs(track_direction - params['heading'])
    if direction_diff > 180:
        direction_diff = 360 - direction_diff
    
    abs_heading_reward = 1 - (direction_diff / 180.0)
    heading_reward = abs_heading_reward * heading_weight
    
    # - - - - -
    
    # Reward if steering angle is aligned with direction difference
    abs_steering_reward = 1 - (abs(params['steering_angle'] - direction_diff) / 180.0)
    steering_reward = abs_steering_reward * steering_weight

    # - - - - -
    
    return speed_reward + heading_reward + steering_reward
{% endhighlight %}

You can see how the car performs during the evaluation phase:
<iframe src="https://www.youtube.com/watch?v=dQ2NinHaC5c" allowfullscreen></iframe>

or click this [youtube link](https://www.youtube.com/watch?v=dQ2NinHaC5c)

Related:
- [How to win AWS DeepRacer](https://medium.com/axel-springer-tech/how-to-win-aws-deepracer-ce15454f594a)
- [DeepRacer docs](https://docs.aws.amazon.com/deepracer/latest/developerguide/deepracer-reward-function-input.html?utm_source=Udacity&utm_medium=Webpage&utm_campaign=Udacity%20AWS%20ML%20Foundations%20Course#reward-function-input-objects_speed)
