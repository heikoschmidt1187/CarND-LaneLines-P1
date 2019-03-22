# **Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[imageBase]: ./examples/01_base.png "Base Image"
[imageGrayBlur]: ./examples/02_gray.png "Grayscale and Gaussian Blur"
[imageCanny]: ./examples/03_Canny.png "Canny edge detection"
[imageRoi1]: ./examples/04_ROI.png "ROI Visualization"
[imageRoi2]: ./examples/05_ROI2.png "ROI image"
[imageHough]: ./examples/06_Hough.png "Hough image"
[imageResult]: ./examples/07_result.png "Resulting image"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

In general, my pipeline consists of six steps. Let's follow them along with this exa[mple image:

![Base Image][imageBase]

The first step for further analysis is to reducy complexity of the three channel RGB image data. This is done by convertig it to a one channel grayscale image. In the same step, to reduce noise, I apply a Gaussian blur image filter with a fixed kernel size of five (worked best with the given images). This is the result:

![Grayscale Blur Image][imageGrayBlur]

In the next step of my pipeline, edges are detected using the Canny edge detection algorithm. The parameters for the Canny function have been determined through a try-and-error approach on all three of the given videos. This gives us the folliging picture:

![Canny Image][imageCanny]

Note that the edges of the lane lines are standing out quite well, but there are many additional contours we don't need and don't want for our analysis. This is faced by defining a specific region of interest (ROI) with respect to the (fixed) mount position of the camera in the car.

The ROI itself is a four point polygone with a perspecitve-like shape. The goal is to have the lines inside the polygone, whereas all not needed information is cancelled out. The polygone looks like this:

![Polygone ROI image][imageRoi1]

From this polygone, a ROI mask is obtained and applied to the Canny image:

![ROI on Canny][imageRoi2]

At this point, we have only the lane lines in the image and need to detect the desired lines in driving direction. To do this first, I use the Hough line transformation to get lines out of the image. These lines need to be separated into left and right of the middle mounting position. The main difference of left and right is the sign of the slope value for each Hough line. A negative slope value indicates a left line, a positive a right line. But watch out - there are small parts in each lane line, that are quite horizontal in each lane. To avoid misinterpretation, the slope values are limited. For this, there is a bias value (-0.65, 0.65) and a delta value (0.15) giving a region in which the slope needs to lie. There is an additional handling for videos, described shortly. The resulting Hough lines look like the following (note: blue = left, red = right, all parameters for Hough and slope have been determined through try-and-error):

![Hough lines][imageHough]

While the detected lines look good, there are too much of them. The goal is to have one solid line left and right, so the Hough lines need to be averaged and extrapolated. So for this, I first averaged the slopes of the lines and then averaged the X- and Y-values for all the lines, giving something like the middle of all detected lines in the image. These values I used in addition to the averaged slope to get the Y-intersection parameter of the line equation y = a * x + b. With the line equation by hand, I used the Y bottom and top position of the ROI polygone to calculate the corresponding X positions on the line.

Again, there is a special handling for videos, described later.

So finally, the image looks like this:

![Resulting image][imageResult]

#### Special handling for videos

Note: I didn't touch the draw_lines() function, but instead put the implementation into own helper functions. The reason for this is the special video handling described below.

The above pipeline worked quite well with static imaged, but had flaws in videos as a sequence of images. As the Hough lines are very noisy from frame to frame, the average values for slope and points varied from frame to frame. This leads to very "restless" jumping lines.

To approach this, I used some kind of history dependency to the detected lanes from the previous frame (if there is any). So the bias slope when deciding for left and right Hough lines is dynamically adapted to the slope of the previous detected line. This ensures that if the change in the slope is too high, the line is not recognised.

This wasn't enough, so I decided to average changes in the X and Y points for each line with the corresponding points of the previous line. This smoothed the lines better.

### 2. Identify potential shortcomings with your current pipeline

There are many shortcomings in the implemented pipeline.

First (and most), it's working terrible in curvatures. Always predicting two straight lines without curvature leads sooner or later to crossing lines that are not reflecting reality. Steep curvatures are also bad.

Second, the lines are still quite restless in videos. Different lighting conditions, different weather conditions and different road conditions leads to changes in brightness, in edge- and line detection.

Third, with the example videos there is no car directly in front. If cars drive into the ROI, there will be detected edges that are not reflecting lane lines and therefore the resulting line will be wrong --> I created several videos driving around in my area, so I will check this in practice when I find the time :-)

Fourth, there is no color filtering. This wasn't really needed for the project, but e.g. here in Germany we have white lanes on normal roads, and yellow lanes indicating construction area/zone lines overlapping white lines. So in this case it's important to ignore white lines and give the yellow lines a higher priority.

Last but not least, the implemented method doesn't give us too much indication about distances. One may estimate it through line gaps, but in general it's not a sharp value. Same with curvature measurement - it's simply not possible with the straight lines and the missing distance information.

Especially in the challenge video nearly all off the shortcomings come to reality.

### 3. Suggest possible improvements to your pipeline

The following improvements may fix some of the above issues:

Issue 1:
Curvatures may be handled by using segmented lines instead of one solid line per side. The segment length needs to be determined carfully.

Issue 2:
A well parametrized PID Filter over time should work well with the problem of restless lines. For the lighting conditions some kind over dynamic brightness leveling might be useful, meaning to do a grayscale thresholding based on the average image brightness for example. Still there will be problems with wet reflective roads. One approach for this may be a detection through machine learning.

Issue 3:
This may be handled through a perspective transform using the calibration matrix of the camera. Also the preprocessing parameters may be tuned better.

Issue 4:
Do a color filtering for white and yellow.

Issue 5:
Same as Issue 3 - use perspective transform and camera calibration matrix.

General:
The Python code is not optimized to extend. There are many "not so beatiful" parts that are working for this project, but in productive should be refactored with respect to code quality, effectiveness and runtime.
