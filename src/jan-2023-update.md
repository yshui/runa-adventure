# January 2023 updates

I have implemented surface scaling and fixed coordinate system conversion between wayland surface coordinates and screen coordinates. So now everything is finally the right size and the right side up:

![demo](assets/jan-2023-update.png)

Behind the scenes, I fixed the implementation of sync'd wayland subsurfaces. The previous implementation was actually completely wrong.

Next, I will try to implement mouse and keyboard input support. Once that is done, I plan to make the repository public.


