# RPG
## A high performance python library for displaying drifting gratings on the Raspberry Pi 3.

## Download
Move to your home directory, and clone the repository

```
    $ cd ~
    $ git clone https://github.com/bill-connelly/rpg
```
    
## Configure
Before installation you need to set the degrees of visual angle your monitor convers. This is a simple edit of a single line of code.

```
    $ cd rpg
    $ nano rpg/_rpigratings.c
```
        
On line 28 the code will say `#define DEGREES_SUBTENDED 80` change the number `80` to however many degrees your monitor covers. Then press `cntrl-x` to exit `y` to save, and press enter to accept file name.

## Install

Confirm you are in the rpg directory (i.e. the directory with setup.py) and install
```
    $ cd ~/rpg
    $ pip3 install .
```
## Run

RPG is designed to run in response to 3.3 volt triggers from other hardware, or in a free running mode, where it will provide a 3.3V output when it displays each frame. There are several examples scripts in the examples folder. But briefly, to confirm that RPG has been install successfully, and to show it's functionality, the following lines of code can be run. 

First open python

```
    $ python3
```
Then import the module
```    
    >>> import rpg
```
Now we need to write some drifting gratings to disk. These can either be fullscreen, in a circular mask or as gabor patches. In this example, we will create a full screen gratings. First we need to create a dictionary to store the parameters we intend to use.
```
    >>> options = {"duration": 2, "angle": 45, "spac_freq": 0.2, "temp_freq": 1}
```
Now we will write the grating to disk, aka, *build* the grating. This grating will have the default resolution of 1280x720, but many other resolutions are possible. To build a grating like this, the directory it is being written to must already exist, so we will write it into the home directory.
```
    >>> rpg.build_grating("~/first_grating.dat", options)
```
You will receive some feedback. Despite requesting a temporal frequency of 1 cycle per second (the last option in the dictionary), you may be told that the temporal frequency of the file not quite what was requested. This change was performed in order to optimize file size and memory usuage and because the wavefront must proceed at an integer number of pixels per frame.

Now we will create a Screen object. This object contains the methods to load and display gratings. We will use the default resolution of 1280x720. The resolution of the Screen object must match the resolutions of the gratings being displayed.
```
    >>> myscreen = rpg.Screen()
```
Now we will load the the grating built earlier. This command returns a grating object.
```
    >>> grating = myscreen.load_grating("~/first_grating.dat")    
```
Finally, we can display the grating
```
    >>> myscreen.display_grating(grating)
```
Instead of displaying the grating as soon as possible, it is also possible to specify to wait until a 3.3V trigger signal is recieved on a GPIO pin. The Raspberry Pi works on 3.3V logic, rather than the 5V TTL logic of most DAQ boards an arduinos, hence we need to step down the voltage level. This can be achieved with a simple voltage divider, but we recommend using a bidirectional logic shifter such as the [BOB-12009 from SparkFun](sparkfun.com/products/12009). We use ths wiringPi library to control the Raspberry Pi GPIO pins, as as such have chosen to use their default, if somewhat unconvention [numbering system](wiringpi.com/pins). WiringPi Pin 1, which is the physical pin 12 on the header, is used to supply feedback, alternating between high and low during each frame update, hence this pin is reserved.

This trigger behaviour is generated by specifying a pin that RPG will listen to, and wait until a trigger is received before displaying the grating. The delay between the trigger and the grating being displayed is between 2 ms and 1000/[refresh rate of monitor] + 2 ms. 

In order to specify to listen for a trigger signal on pin 6 (physical pin 22 on header):
```
    >>> myscreen.display_grating(grating, 6)
```

## More examples

RPG is designed to be flexible, howevever, we believe most users will find the `build_list_of_gratings()` the most useful way to build gratings and the `Screen.display_gratings_randomly()` or `Screen.display_rand_grating_on_pulse()` methods the most useful way to display them.

RPG allows the user to iteratively build lists of gratings simply by specifying the options dictionary to contain one setting as a list. RPG will then iterate through this list, building gratings that are identical, apart from the parameters specified in this list. For example, if a user wanted to generate circular masked gratings at 12 different orientations, the options dictionary could be specified as
```
   >>>  options = {
        "duration": 2,
        "angle": [0,30,60,90,120,150,180,210,240,270,300,330],
        "spac_freq": 0.2,
        "temp_freq": 1,
        ...
```
Because this is a grating with a circular mask, we also need to specify it's diameter (as percentage of screen width) and position (as percentage of screen width and height):
```
        ...
        "percent_diameter": 30,
        "percent_center_left": 50,
        "percent_center_top": 50
         }
```
Now we pass that option dictionary to the function `build_list_of_gratings()` along with a string representing why type of grating with want (either "grating", "mask" or "gabor"), and the directory we want to build all our gratings in. We intend the user to place gratings to be displace in sequence in the same directory. This way that can be conviently loaded and displayed.
```
    >>> rpg.build_list_of_gratings("mask", "~/gratings/variable_ori/", options)
```
This builds gratings at each of the specified orientations, but matching in all other regards. In this manner, any property that can be specified in the option dictionary can be itterated through.

Now we need to display the gratings. We can use either the `Screen.display_gratings_randomly()` or `Screen.display_rand_grating_on_pulse()` methods to display these gratings itteratively, the first, displaying them at a fixed interval, and the second in response to a 3.3V stimuli to a specific pin.  

In order to iterate through gratings in response to a 3.3V trigger to pin 6 (physically pin 22 on header) call
```
    >>> myscreen.display_rand_grating_on_pulse("~/gratings/variable_ori/", 6)
```
The performance record of this will be recorded, by default, in ~/rpg/logs/rpglog.txt. This logfile saves the output in a tab separated file, where each line is a displayed grating. The elements in each row are, filetype ("grating" or "raw"), start time (in unix time), average frame duration (microseconds) and the standard deviation of the frames displayed (microseconds)

## Raws

RPG is capable of displaying images and movies with the same temporal accuracy as the gratings. However, in order for these to be loaded and played efficiently, the must be converted into a format suitable for RPG. Furthermore, because of the vast number of image and movie formats available, we decided that it is the users responsibility to get the code into a raw format first.

Specifically, the file must be saved pixelwise from the top left pixel, proceeding row-wise, with each pixel saved as (uint8) R, (uint8) G, (uint8) B.... until the final bottom right pixel. If the file contains more than one frame, i.e. it is a movie, then the subsequent byte saves is the red value of the top left pixel of the second frame. We have included some examples of how to process videos and images in the examples folders.

Once a raw file is produced to the above specification, it can be converted to a format suitable for RPG to play with the function `convert_raw`. Specifically, if a raw 30 FPS movie at 1024x768 resolution with 200 frames is saved in "\~/import/rawmovie.raw" it can be converted and saved to "\~/raws/raw_c.raw" with the following
```
    >>> convert_raw("~/import/rawmovie.raw", "~/raws/raw_c.raw", 200, 1024, 768, 2)
```
The last argument is the number of monitor refreshes per frame. Specifically, in order to maintain accurate and reproducable timings, typical movie frame rates of 29.97 or 25 cannot be specified directly. Instead, an integer multiple of the duration of a monitor refresh rate is specified. Thus, in order to achieve 30 FPS, the monitor refresh rate should be set to 60, and the final argument should be 2. In order to achieve 25 FPS the monitor refresh rate should be set to 50 Hz, and the final argument 2, or the monitor set to 100 Hz, and the final argument 4.

The second argument, the number of frames, should not be used to clip movies. The entire movie will be converted if this number is set to less than the duration of the movie on disk, however, only the specified number of frames will be played.

Movies can take up significant amounts of memory, e.g. a 400 frame, 1024x768 movie will take 16*1024*768*400 bits or 629 MB, which is practically the entirety of the free memory. This means multiple movies are not able to be stored in RAM simultaneously. This should be considered when designing experiments.

Images can be converted just the same as movies, except one specifies the number of frames as 1, and the last argument as the duration the image should be displayed in monitor refreshes, e.g. if an image is to be displayed for 1.5 seconds, on a 60 Hz monitor, this argument should be entered as 90.

If a raw image is saved in "~/images/1_c.raw" it can be loaded into memory with:
```
    >>> raw = myscreen.load_raw("~/images/1_c.raw")
```
This can then be displayed with:
```
    >>> myscreen.display_raw(raw)
```
In a similar fashion to drifting gratings, raws can also be displayed in response to a 3.3V pulse by specifying the pin number:
```
    >>> myscreen.display_raw(raw, 6)
``` 
Likewise, analogous functions to `display_grating_randomly()` and `display_rand_grating_on_pulse()` exist for raws. You can randomly select raws from a directory, and display them with an intertrial time of 2 seconds:
```
    >>> myscreen.display_raw_randomly("/~/raw_directory/, 2)
```
Or you can display a random raw from a directory in response to a 3.3V pulse with:
```
    >>> myscreen.display_rand_raw_on_pulse("/~/raw_directory", 6)
```



Tested on Raspian GNU/Linux 8, Python 3.4.2.

## Troubleshooting

RASPBERRY PI 4
    The Raspberry Pi 4 is a newly released version of the raspberry pi that supports
    dual monitors and 4GB of memory. We believe both of these features will improve
    the usability of RPG significantly. RPG does not function on this hardware at
    present due to a significant change in how graphics are displayed. Until an
    update is made, only use RPG on Raspberry Pi model 3.

STATIONARY GRATINGS
    Low resolution combined with low propogation speeds (low temporal frequency or
    high spacial frequency) and high FPS may result in unmoving gratings, because the
    per-frame propogation speed is rounded to the nearest integer number of pixels
    (such that propogation is even each frame), and this speed may be rounded down to zero.

MEMORY MANAGEMENT
    The raspberry pi 3 has 1GB of memory, of which typically 750MB is free. Each 1280 x 720
    frame of 16 bit data takes up 14.75MB of ram. A typical 1 seconds grating with no looped
    frames at 60 Hz therefore takes up 110.6MB. Hence a maximum number of *unlooped* gratings
    that can be stored is less than 7. However, RPG attempts to make gratings loop, which is 
    achievable at high temporal frequencies, e.g. a 1 second grating at 2 cycles per second 
    only requires half the frames, as the second cycle is identical to the first. However, gratings
    where the temporal frequency is less than the duration will require every frame within it 
    generated, and stored to memory. This may lead to crashes if many of these are loaded at once.
    If the user requires many gratings at less than 1 cycle per second to be loaded simultaneously,
    it is likely that RPG will not be suitable until version 2 is releated that will support the
    Rasbperry Pi 4. 
