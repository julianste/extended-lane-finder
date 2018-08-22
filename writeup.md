## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

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

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./examples/distortion_correct.png "Distortion Corrected"
[yellow_white]: ./examples/yellow_white.png "Yellow White Image"
[unmasked]: ./examples/unmasked.png "Unmasked Image"
[masked]: ./examples/masked.png "Masked Image"
[warpedStraight]: ./examples/warpedStraight.png "Warp straight line"
[image5]: ./examples/color_fit_lines.png "Fit Visual"
[image6]: ./examples/example_output.png "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/julianste/extended-lane-finder/blob/master/writeup.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the third code cell of the IPython notebook located in "./P2.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

I implemented a class called `ExtendedLaneProcessor` with methods for each of the following steps. You find the implementation in the cell with title "Lane Processing Pipeline" in `./P2.ipynb`

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images. The following image is distortion corrected:

![alt text][image2]

I called the function `cv2.undistort` with the parameters I calculated by use of `cv2.calibrateCamera` in the preceding step

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Similar to the first project I sticked with different color thresholds for this part. In the end I didn't use any gradients as I had no success with integrating them into my pipeline. My pipeline towards a thresholded binary image is composed of the following steps:

1. Extract yellow and white colors by first converting to HSV and then using cv2.inRange with predefined yellow and white lower and upper bounds. 

![alt text][yellow_white]

2. From this yellow-white image, extract the following channels: **s,g,b,r**
3. We want to have two images: one white and one yellow: 

		high_yellow = np.logical_and(high_r, high_g)
        high_white = np.logical_and(np.logical_and(high_r, high_g), high_b)
		
4. our (unmasked) binary image has a 1 wheneither *high_white* OR (*high_yellow* and *s_high*) are activated. In other words:

`unmasked = np.logical_or(high_white, np.logical_and(high_yellow, s_high)).astype(np.uint8)`

After this, the binary looks like:

![alt text][unmasked]

5. mask the image for the relevant region

![alt text][masked]

6. For the video pipeline I aggregate this binary with the binary from $n$ previous frames (usually $n=2$). The reasoning behind this is that the lane markers wont change that quickly during 2-3 frames and we can use the information from the previous frames towards a more stable lane detection.


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.
In my pipeline class, the function `apply_perspective_transform` does the transformation. I hardcoded the src and dst points as follows:

```python
        src = np.float32(
             [[230, 685],
             [586, 455],
             [695, 455],
             [1043, 685]])
        dst = np.float32(
             [[(img_size[0] / 4), img_size[1]],
             [(img_size[0] / 4), 0],
             [(img_size[0] * 3 / 4), 0],
             [(img_size[0] * 3 / 4), img_size[1]]])
```

This resulted in the following source and destination (x,y)-cooridnates:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 230, 685      | 320, 720      | 
| 586, 455      | 320, 0        |
| 695, 455      | 960, 0        |
| 1043, 685     | 960, 720      |

I verified that my perspective transform was working as expected by applying it to images with straight lanes and verifying that it was straight in the bird's eye perspective.

![alt text][warpedStraight]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I identified the lane-line pixels by the convolution approach from the lecture, finding the activated pixels and fitting a second order polynomial via `np.polyfit`. There were two issues I was facing:

1. The windowing approach fails if there are no lane lines in the bottom quarter of the picture since then no initial bottom windows can be found. In this case I extend the vertical summation to the bottom half of the picture. If also there is no lane line found, we just use the polynomials from the last frame and hope to find the lanes in the next frame
2. In case we cant find any activated pixels for one of the sides, we also use the polynomial from the last frame.

Although theses issues happened, they are very rare as we always use the last three frames together in order to determine the lane lines. It is reasonable that in some of those frames lines are present and can be detected.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in functions `measure_curvature_real` and `get_distance_from_center` respectively. The radius and curvature are only updated every 5 frames. While the calculation of the deviation from the center of both lanes to center of the image gives the position of the vehicle the radius is a bit more tricky: We compute it for the left and the right lane and calculate the average.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step a seperate cell after the implementation of my class in `P2.ipynb`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](https://github.com/julianste/extended-lane-finder/blob/master/video_output/project_video_final.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

While my pipeline does pretty good on the project video, for the two challanges my lane prediction is not very stable. Especially in the hard challenge, all the different light effects and the extreme curvings make it very difficult. In general it is difficult to come up with a "one-fits-all" pipeline. For example, in the hard challenge video we currently mask out too much regions. we would also have to deal with the case when one of the lines is not visible at all.

Currently, whenever one of the lane lines (left or right) can not be determined, we just use that from the last frame. This assumes that at some point (a few frames later at most) our system is able to recover this line.

Another idea to make it more robust: Discard a polynomial line if it differs (in some sense) too much from the (averaged) previous lines. It is likely that in this case our prediction of the line will be wrong.



