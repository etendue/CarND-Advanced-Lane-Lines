
**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/Original_Undistorted_scene.png "Undistorted"
[image2]: ./output_images/threshold_filters.png "Threshold filters"
[image3]: ./output_images/Perspective_transform.png "Perspective transformation"
[image4]: ./output_images/Histogram.png "Histogram"
[image5]: ./output_images/Detect_lanes.png "window sliding"
[image6]: ./output_images/Search_lanes.png "Search lanes"
[image7]: ./output_images/warped_lanes.png "Warped lanes"
[video1]: ./output_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  
You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md)
 is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the code **cell 3** of the IPython notebook located in "./project4.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. 
Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each 
calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time 
I successfully detect all chessboard corners in a test image. `imgpoints` will be appended with the (x, y) pixel position 
of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using 
the `cv2.calibrateCamera()` function. I applied this distortion correction to the test image using the `cv2.undistort()` 
function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one.
The bottom two pictures show the image before and after distortion-correction.



#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image. Provide an example of a binary image result.
I used a combination of color and gradient thresholds to generate a binary image. From code **cell 4 - 8**, I create following 
functions to implement different threshold filters.
```python
abs_xy_thresh(img,orient='x', sobel_kernel=3,thresh=(0,255))
mag_thresh(img,sobel_kernel=3,thresh=(0,255))
dir_thresh(img,sobel_kernel=3,thresh=(0,np.pi/2))
hls_saturation_thresh(img,thresh=(0,255))
```

After I have tweaked with the parameters I packed the individual threshold filters into a function in code **cell 13**.
```python
process_img_with_filters(img, x_fil=None,y_fil=None,m_fil=None,a_fil=None,s_fil=None)
```

Here's an example of my outputs after different threshold filters.
(Note: the image is after the perspective transformation explained in next section)


![alt text][image2]


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform are in code **cell 9-10**. In **Cell 9** I picked four points from the left and right lanes. Assuming these four points constitute a rectangle. 

| Source        | Destination   | Position   |
|:-------------:|:-------------:|:----------:|
| 247, 691      | 250, 720      | left bottom|
| 601, 447      | 250, 0        | left top   |
| 679, 447      | 950, 0        | right top  |
| 1061, 691     | 950, 720      | right bottom|


The choice of destination points affects which parts of the original image to be mapped to new image. I adjusted them 
several times to keep mostly the lanes but removing the trees and sky if possible.

I verified that my perspective transform was working as expected by drawing the a test image with `straight_line1.jpg` and its warped counterpart to verify that the lines appear parallel in the warped image.


![alt text][image3]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

##### 4.1 detect lane from scratch

To detect lane line, I take the following procedure:
 1. take a histogram along x axis(y values are counted) of binary image from above steps (threhold filtering and perspective transformation)
 2. locate two peaks of at left side and right side of histogram(these two peaks are assumed as lanes position)
 3. slide a window along y axis to capture lane pixels 
 4. ployfit the captured lane pixels to get 

This procedure is implemented in code **cell 16** with function name:
```python
# use window sliding to search the lane from binary image
def detect_lanes(bin_img,plot=False):
```
Here is an illustration of histogram of image and how  sliding works:


Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image4]![alt text][image5]

##### 4.2 search lane based on previous detection, LOOK AHEAD

Since the lanes will not jump from frame to frame so quickly in a recorded video, we can assume the lanes remain the same 
position with slightly change. So next function searches only the lanes pixels nearby (left and right margins marked with green)where lanes were found in previous image frame. See following picture to see:
 * the search window (place of interest) is marked with green
 * the found pixels(red pixels at left and blue at right)
 * polyfitted lines.

![alt text][image6]

It is interesting to pay attention that the white area within the green region. The lane pixels are not found by search 
algorithm but the lane line after polyfit is still satisfying somehow. The search_lanes function is implemented in code **cell 19**.

```python
# from the next frame of video
# It's now much easier to find line pixels!
def search_lanes(bin_img,previous_lanes=None,plot=False):
```

##### 4.3 Sanity check

The detected lane lines are not always correct due to shortage of applied parameters of gradients and color threshold 
filters or applied technique itself. So Sanity check does the verification if detected lanes are confident.

Code **cell 23** implements the sanity check.
    1. Checking that they have similar curvature
    2. Checking that they are separated by approximately the right distance horizontally
    3. Checking that they are roughly parallel

The used down/up limits are not well investigated and only tested against the test images.
Here I use, 20% for curvature different, 2.8m and 5m for distance between lanes, and 0.5m deviation of of distances 
between two lanes.  If detected lanes lines are over the defined range, the sanity check will return False and they
are rejected as confident lanes. 


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

To calculate the radius of curvature and vehicle position (see code **cell 21** )to lane at both sides, the pixel coordinates have to be converted to physical world coordinates. I used the hard coded estimated conversion:

```python
    # Define conversions in x and y from pixels space to meters
    ym_per_pix = 30/720 # meters per pixel in y dimension
    xm_per_pix = 3.7/700 # meters per pixel in x dimension
```

##### 5.1 radius of curvature of the lane
Then the converted lane points are used to polyfit the lane line to get polynomial coefficients. The curvature is 
calculated based on following formula:


$$R_{radius} = \dfrac{(1+(2Ay+B)^2)^{3/2}}{|2A|}$$

(why github can not display the math latex???)


$A,B,C$ are polynomial coefficients for lane line


$$f(y) = Ay^2 +By +C$$



##### 5.2 position of the vehicle with respect to center

We assume the distance between left lane and right line is 3.7m and vehicle is always centered in the image. So by 
calculating the lane x coordinate to image center we can calculate how far is vehicle center away from left and right 
lane respectively.

To check if the vehicle is centered, we compare the distances/offsets of vehicle center to left/right lanes. If both 
 have same value, then vehicle is centered, small offset to left means vehicle is shifted to left side and vice versa.
 
The curvature and offset to lanes are output in processed video at left top corner of image with following format:

``` python
"Curv.:L686.5m,R683.2m"
"Offset: to LLane:1.7m | to RLane:1.9m"
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in code **cell 25** with function `warp_lanes_back()`.  I used the code from lecture and modified 
some lines to add additional information like put text of lane curvatures.

The function `warp_lanes_back()` mainly does the following jobs:
* draw the ploygon of detected lanes
* do a perspective transformation 

Here is an example of my result on a test image:

![alt text][image7]

---

### Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

The pipeline to process the video is implemented in code **cell 27**. Here the strategy for LookAhead and Reset is used to skip the "bad frame". The "smoothing" is also implemented by averaging the current polyfitted line coordinates with previous detected line. 

Here's a [link to my video result](./output_video.mp4)


[video1]

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

#### Problem 1: how to use the gradient threshold filters
There are gradients on x, y, magnitude and direction filters, not all work well on same image, so as color filters.
At moment a fixed combination of filters with fixed parameters are used. The filter may work on the project video but fails
quickly with challenge video and harder challenge video. 

Since individual filter excel at certain scenario, I wonder I can use them separately and dynamically, i.e. 
* use each of filter separately and parallel and pick the best result for lane detection from them by sanity_check. 
* if none is satisfactory, then adjust the parameter dynamically.

These shall increase the chance to get a reasonable confident detection at cost of more calculation

#### Problem 2: Smoothing
At moment when lane search/detection fails to get a confident result, I just use the last good detection. Thing goes bad
after continuous no good results, a good result is captured, then rendering of detected lanes will jump in video. By 
averaging the lines position with previous lane will mitigate this jumping a little. I used only averaging the current 
detection with last one. To get a better result, more experience may be required for adjusting strategy and parameters.
(Somehow the effect is only seen in video and working on video needs some patience, 10 minutes to wait :( )

