## Advanced Lane Finding Project
----

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

[image1]: ./output_images/undistorted_chessboard.jpg =650x450 "Undistorted"
[image2]: ./output_images/undistorted.jpg =650x450 "Undistorted"
[image3]: ./output_images/binary.jpg =650x450 "Binary"
[image4]: ./output_images/warped_binary.jpg =650x450 "Warp"
[image5]: ./output_images/lanes_polynomials.jpg =650x450 "Fit Visual"
[image6]: ./output_images/projected.jpg =650x450 "Output"
[video1]: ./project.mp4 "Video" =650x450


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  


This is the writeup. 


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.


The code for this step is contained in the section named "Step 1" of the IPython notebook located in "./P4.ipynb".

The starting point is to prepare the object points, which will be the (x, y, z) coordinates of the corners of the chessboard in './camera_cal'. It is assumed that the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objPoint` is just a replicated array of coordinates, and `objPoints` will be appended with a copy of it every time all chessboard corners in a test image are successfully detected. `imagePoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

Then, the output `objPoints` and `imagePoints` are used  to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function. This distortion correction is applied to the test image using the `cv2.undistort()` function. An example output is:

![alt text][image1]


### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.


The camera calibration and distortion correction step above is applied to one of the test images. Although it is not really clear if the test image is undistorted just by looking at it, it should be correctly undistorted because the same step is applied to it as above:


![alt text][image2]


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code for this step is contained in the section named "Step 2" of the IPython notebook located in "./P4.ipynb".

A combination of color and gradient thresholds is used to generate a binary image. See a function called `thresholdColorGradient()`.

For thresholding by color, the undistorted image is converted into HLS color space and separated into the lightness channel and the saturation channel. For thresholding by gradient, the Sobel operator is applied to the lightness channel to take the derivative in x dimension. 

The lightness channel applied by the Sobel operator and the saturation channel are filtered by thresholds, and then combined. 


![alt text][image3]


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for this step is contained in the section named "Step 3" of the IPython notebook located in "./P4.ipynb".

The code for the perspective transform includes a function called `warp()`. The `warp()` function takes as inputs an undistored image. The hardcoded source and destination points are chosen in the following manner:

```
srcUpperLeft = [(imageSize[0] / 2) - 60, imageSize[1] / 2 + 100]
srcLowerLeft = [((imageSize[0] / 6) - 5), imageSize[1]]
srcLowerRight = [(imageSize[0] * 5 / 6) + 55, imageSize[1]]
srcUpperRight = [(imageSize[0] / 2 + 85), imageSize[1] / 2 + 100]

dstUpperLeft = [(imageSize[0] / 4), 0]
dstLowerLeft = [(imageSize[0] / 4), imageSize[1]]
dstLowerRight = [(imageSize[0] * 3 / 4), imageSize[1]]
dstUpperRight = [(imageSize[0] * 3 / 4), 0]
```

The `src` and `dst` points are mapped onto a undistorted binary image and its warped counterpart to verify that the lines appear almost parallel in the warped image. The transformed lanes are not perfectly parallel. However, this is a result of tuning of the hyper parameters of the `src` points. 

The inverse perspective matrix `inversePerspective` is also generated here for creating an image of lane boundaries. 


![alt text][image4]


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for this step is contained in the section named "Step 4" of the IPython notebook located in "./P4.ipynb".

As in the lecture, the sliding window was used to identify lane-line pixels and fit their positions with a polynomial. See a function called `findLanes()`. 

First of all, find the peak of the number of pixels in the left and right halves of the image utilizing a histogram, then find nonzero indices of pixels within a window by sliding the window from the bottom to the top. The nonzero indices are converted into pixel positions and coefficients of a polynomial is calculated using `numpy.polyfit()` and the pixel positions. This 'full scan' step is done for the first frame of a video. 

Once the coefficients of a polynomial are calculated, these values are leveraged to calculate the nonzero indices of pixels, and then updated again. This step is repeated for the rest of frames of a video. 

In order to avoid accepting anormalies of the frame image, `validated()` and `smooth()` functions are introduced. 

`validated()` is a function to check if the x-intercepts of the polynomials and the radii of curvature get anormal values comparing to the ones of the previous frame. If those values do not pass the criteria of the validation, the values of the previous frame are used to draw the lane boundaries. If the validation does not passed successively specific times, then the 'full scan' step is conducted in the next frame.

`smooth()` is a function to smooth the lane lines by averaging them over consecutive frames. 


![alt text][image5]


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code for this step is contained in the section named "Step 4" of the IPython notebook located in "./P4.ipynb".

As in the lecture, the formula is used to calculate the radius of curvature of a lane after coefficients of a polynomial is calculated by `numpy.polyfit()`. See a function called `measureCarvature()`.

Regarding the position of the vehicle, the vertical and horizontal center is calculated as the center first, then the position of the vehicle is calculated with respect to the center so that it becomes a positive value when the vehicle is right of the center, and vice versa.

Another way to calculate the center would be to use the x-intercept returned by `numpy.polyfit()`. However, this was not used because the x-intercept of a polynomial fitted to a curved lane tends to include errors. Assuming the lanes are parallel, using vertical and horizontal center would be valid. 

Both radius and the position of the vehicle is calculated in meters. In order to calculate them in that way, the meters per pixel for x and y dimension had to be manually set by checking the transformed image to see how lanes were placed in the image. 


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The code for this step is contained in the section named "Step 5" of the IPython notebook located in "./P4.ipynb".

See a function called `projectLanes()`. Using the inverse perspective transform matrix `inversePerspective`, `cv2.warpPersvective()` does the inverse transformation. 

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

This is a link to [the final video output](./project.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?


As in Project 1, a problem I faced was to set a proper region to be processed in an image. In this project, that corresponds to the step to set the source and destination region for the perspective transform. Setting the regions were done essentially by trial-and-errors. Since this step also affected the number of pixels that lanes cover, this step had to be done deliberately. 

This step would definitely have to be improved. I noticed the suggestion in the lecture saying "many perspective transform algorithms will programmatically detect four source points in an image based on edge or corner detection and analyzing attributes like color and surrounding pixels." This would be explored after this project is safely done for the minimum requirements. 

Tuning the hyper parameters of the validation also took me a long time. Especially, setting a proper threshold of the difference for the x-intercepts of the polynomials was difficult. If it was too small, the chance to recover from anormalies of the x-intercepts would get smaller, then the shape of the lane lines could get stuck. If it was too large, the chance to accept anormalies of the x-intercepts would get larger, then the shape of the lane lines could get unstable. The region to be processed would also affect the hyper parameters of the validation. An adaptive way to set those hyper parameters would also be necessary.

I eventually decided to fully scan every frame to get a result though the algorithm to detect lanes is implemented to leverage measurements from the previous frame. I did so because the video produced looked better than leveraging measurements from the previous frame. 

  
