# Lowering click-to-photon latency by eliminating the render queue

osu!lazer has an amazing rendering and threading system that allows the user to configure the game to their liking and hardware specification.
Yet, currently, there is only one possible scenario in lazer to achieve the most responsive click-to-photon latency (I will refer to click-to-photon latency as input latency from now on, a naive abstraction but it's simpler to read. Perceptually, they mean the same thing).

The optimal scenario for lowest input latency in lazer is one where the user limits their framerate to 1000fps, even if multithreading is already polling your inputs at 1000hz on any frame limiter. Why is this? It's because the final bottleneck of the rendering pipeline is the render queue. There will always be a render queue in lazer, you can limit it to 1 frame via the nvidia control panel's "low latency mode", but there will always be a queue of at least 1 frame.

In the best case scenario where the rendere queue is equal to 1 frame, the time it takes to get through this queue is equal to 1 frametime. In the case when you are playing on a 240hz monitor, drawing 240fps would result in the render queue being a bottleneck of 4.17ms (This is the frametime of 240fps, since the most recent frame is always queue'd behind the currently displayed one). For this reason alone, capping your framerates while playing with multithreading enabled still is NOT a good way to play the game.

We can verify this by injecting Reflex latency markers into lazer with Rivatuner Statistics Server.

![image](https://github.com/user-attachments/assets/74c890f9-2b17-4ef9-bd5f-b147e97218ed)

Running the game at 780fps, the game's "sim-to-render" latency is fluctuating between 1ms-1.5ms. This means that the information being rendered is old by the length of one frametime.
This is why FPS is still king in osu!lazer, despite it's multithreaded capabilities.

The goal here is simple, eliminate the render queue. And, fortunately, the technology behind this theory already exists and is widely available across a variety of games by using NVIDIA Reflex.

NVIDIA Reflex employs a technique called "just-in-time rendering", this basically comes down to the GPU only rendering a frame RIGHT BEFORE the next presentation interval. This way, when the CPU and GPU are coordinating well with eachother, you can always display the most recent information calculated by the game, on any framerate. And since osu is already polling our inputs at 1000hz regardless of framerate, this technique is a no-brainer. Even more so when you consider the fact that rendering a single frame can take as low as a fraction of a millisecond in osu.

But would this even work? I thought reflex was a glorified frame limiter that only reduces latency in the case of a GPU bottleneck?

Well, why don't we test it! We can use a third party application like Special K to inject NVIDIA Reflex to give Direct3D the instructions to use this "just-in-time rendering" rendering technique. And the results speak for themselves!

![image](https://github.com/user-attachments/assets/77412fcf-d646-42b3-80d6-8851cb908e29)

On a 390hz monitor, with a 390fps framecap using the Reflex frame limiter, we are achieving lightning fast sim-to-render speeds and near instant present latency! This means that, even on vsync'd framerates, we are rendering frames based on the last possible received information from osu's 1000hz input thread. The result is a smooth, near tear-free, low latency and optimized gameplay experience. Not to mention that with the addition of reflex, the scenario where the user wants to use VRR is also basically solved, as reflex will adjust the present timings and frame rates for the optimal low latency experience there too.

Another fun little feature of NVIDIA Reflex would be having access to latency indicators calculated by the injected markers. So you can replace that naive little frametime indicator in the bottom right of the screen with an actual measurement of sim-to-present latency! This way, players can see the actual latency that is affecting their gameplay.

# Additional notes
## What about lower refresh rates
Here's a test setting my refresh rate to 60hz.

![image](https://github.com/user-attachments/assets/92734cba-7d7d-4f1c-b668-eca3290a71eb)

Using the just-in-time rendering method, we can finally utilize multithreading to it's full extent and reap the benefits even on low refresh rates! The input feels responsive and the presented frames on screen will always show the latest possible information.
This is an absolute game changer for people with weak hardware.

## Wouldn't it be possible to just use any external frame limiter?
No, any external frame limiter that doesn't have some control over the rendering pipeline will introduce additional latency of at least one frametime.

## What about using the in-game vsync? That would be an in-engine fps cap to your refresh rate right?
Well, the results are interesting. The frames themselves are actually rendered with the last received data from the 1000hz input thread, but the vsync tear control overhead introduces 1 frame of present latency. Even in the best possible scenario, it means that this method is just as bad as the other ones.

![image](https://github.com/user-attachments/assets/3dd9f030-3125-4b7e-a45e-913545be22c3)

Note how the render queue length and present latency are increased because of the focus on v-blank synchronization over latency.

## What about platforms outside of NVIDIA?

Special K actually has another mode called their "low latency" frame limiter, which achieves basically the same concept. It could be possible to implement a vendor agnostic native just-in-time frame limiter in osu!lazer maybe? I've just been talking about NVIDIA Reflex as the primary example, since this is tried and proven tech which is already available in the reflex SDK.

# Closing thoughts

I think this is the perfect direction for osu!lazer and would love to see this being officially supported in the game one day.

Quick question... Please let me know if the use of Special K is bannable in lazer :) As it can inject various tools to control the rendering pipeline, I'm afraid I'll somehow get banned if I use it for online play.
