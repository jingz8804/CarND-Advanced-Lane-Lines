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

[image1]: ./write_up_images/undistortion_example.png "Distorted and Undistorted"
[image2]: ./write_up_images/undistortion_example_test_image.png "Test Road Image Distored and Undistorted"
[image3]: ./write_up_images/gradient_based_thresholding_good.png "Gradient Thresholding Good Example"
[image4]: ./write_up_images/gradient_based_thresholding_bad.png "Gradient Thresholding Bad Example"
[image5]: ./write_up_images/color_based_thresholding.png "Color Based Thresholding"
[image6]: ./write_up_images/bird_eye_view_1.png "Bird Eye View"
[image7]: ./write_up_images/bird_eye_view_all.png "Bird Eye View All"
[image8]: ./write_up_images/lane-pixels-window.png "Lane Pixel Window"
[image9]: ./write_up_images/lane-pixels-next-frame.png "Lane Pixel Next Frame"
[image10]: ./write_up_images/draw-lane-on-image.png "Draw lane on image"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook [advanced-lane-line-detection.ipynb](https://github.com/jingz8804/CarND-Advanced-Lane-Lines/blob/master/advanced-lane-line-detection.ipynb).  

The whole process is done by reading through each chessboard image and mapping the corners (image points) in the real world to the fixed ideal positions (called the object points) ranging from (0, 0, 0) to (8, 5, 0). 

This is achieved by using the `findChessboardCorners` method to find the corners' pixel locations in the image. These points, together with the fixed object points are fed into the `calibrateCamera` method which returns the camera calibration and distortion coefficients. We can apply them to new distorted images with `undistort` method to obtain undistorted images. 

As an example, the following two pictures show the undistortion result before and after: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines # through # in `another_file.py`).  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

I first explored a few gradient-based thresholding methods without the color transformation. It worked on some images like this one:

![alt text][image3]

However, this method alone does not always work. It failed to detect the lane line on images like this one:

![alt text][image4]

My assumption is that the yellow lane is barely detected possibly because of the sunlight is so strong there. So we will probably need to use the color transform to get a better result, which is proved to be useful as shown in the following image:

![alt text][image5]

So the result looks very promising. We could try to combine both the gradient and color based thresholding techniques to detect lane lines in various conditions. 

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The bird-eye view transformation is done by the unwarp function in the [Perspective Transform Section](https://github.com/jingz8804/CarND-Advanced-Lane-Lines/blob/master/advanced-lane-line-detection.ipynb#Perspective-Transform)

It takes in two sets of points, one source and one destination and returns the unwarped image, warping factors and the reversed factors. The points are selected manually on a straight line image. 

```python
height,width = undistorted_example_img.shape[:2]

src = np.float32([(550, 480),
                  (740, 480),
                  (258, 680),
                  (1050, 680)])

dst = np.float32([(450, 0),
                  (width - 450, 0),
                  (450, height),
                  (width - 450, height)])
```

I added this function to the previously undistorted image and got a bird eye view of the lane as follows:

![alt text][image6]

Now that the transformation works, I added to the pipeline and applied to all images in the test folder and got the following result:

![alt text][image7]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The polynomial fit was done by using the `sliding_window_polyfit` function and the `polyfit_using_prev_fit` function. Using the histogram of the processed images from the pipeline, we identified possible starting points for the sliding window in the image and configured a margin to find non-zero pixels. The margin size is something I tuned a few times and found 60 to be a good number. Anything larger than that, it could include noisy pixels from cars in other lanes. We move the window up to gather all possible lane pixels. 

![alt text][image8]

Once we detected the lane in one frame, we can use that information to simplify the detection on the next frame by using the `polyfit_using_prev_fit`. Of course, it is based on the assumption that the lines on the next frame wouldn't change drastically, otherwise the polynomial coefficients extracted from the previous frame wouldn't work. The following image shows that based on the lanes detected from the image above, we found the lane in the next frame (or at least some frame a couple of seconds later).

![alt text][image9]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The curvature calculation and distance from lane center is done in the `calculate_curvature_radius_and_center_dist` function. The convertion is done based on the US regulations on the lane width and dash line length. Since the image I chose has only 1 dash line with approximately 100 pixels, I used the following formula to convert the pixels to actual metres. 

```python
# There is only one dashed line on the right lane line which takes up about 100 pixels
ym_per_pix = 3.048/100 # meters per pixel in y dimension

# The width between the two lane lines are about 380 pixels
xm_per_pix = 3.7/380 # meters per pixel in x dimension
```

Now given the polynomial fit of the lane lines in the image and the line pixel positions, we can find the pixel positions of the lanes in the image. We then convert them into metre space and re-apply the polynomial fit to get the real world fit. Then based on the formula below to get the radius:

```python
curve_radius = ((1 + (2*fit[0]*y_0*y_meters_per_pixel + fit[1])**2)**1.5) / np.absolute(2*fit[0])
```

The car position is defined as the `image x midpoint - lane center position`. The lane center position can be calculated by taking the average of the following two points:
- Left fit polynomial intercepted at y = height
- Right fit polynomial intercepted at y = height
We will also need to convert the difference into the metre space by multipling the convertion factor `xm_per_pix`.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the [Draw detected lane line on the original picture](https://github.com/jingz8804/CarND-Advanced-Lane-Lines/blob/master/advanced-lane-line-detection.ipynb#Draw-detected-lane-line-on-the-original-picture) section.  Here is an example of my result on a test image:

![alt text][image10]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)
Here's a [link to my challenge video result](./challenge_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

One small but tricky problem I had was the python variable namespace that really gave me a hard time. The variables defined outside the function can be accessed inside the function without being an argument. This messed up some plottings and took me a while to figure out. 

I solved it by adding more arguments to the function but in the end, to make the video processing work, I have to get rid of them again...This is rather annoying. Is there a better way to do this in Python?
