## Advanced Lane Finding


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

[image1]: ./output_images/calibration1_distorted.jpg "Originial"
[image2]: ./output_images/calibration1_undistorted.jpg "Undistorted"
[image3]: ./output_images/TestImage_distorted_example.jpg "Test image originial"
[image4]: ./output_images/TestImage_undistorted_example.jpg "Test image undistorted"
[image5]: ./output_images/TestImage_binary_example.jpg "Binary Example"
[image6]: ./output_images/Image_pre_persepect_trans.jpg "before Warp"
[image7]: ./output_images/Image_pos_persepect_trans.jpg "after Warp"
[image8]: ./output_images/sliding_window_search_example.jpg "Sliding window search"
[image9]: ./output_images/search_around_poly_example.jpg "Search around previous detected line"
[image10]: ./output_images/inverse_transform_example.jpg "Inverse Transform"


[video1]: ./project_video_output.mp4 "Project Video Output"

[video2]: ./challenge_video_output.mp4 "Challenge Video Output"


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second and third code cells of the IPython notebook located in "./P2.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]
![alt text][image2]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image3]
![alt text][image4]

The effect of undistortion is not big, but one can clearly notice the difference by looking at the position of the white car relative to the right edge of the image.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image. For the color thresholds, I used R, G, L, and S chanel, while for the gradient thresholds, I used a combination of x and direction gradients. I also applied a trapezoidal mask to isolate the lane lines. Here's an example of my output for this step.  

![alt text][image5]



#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is clearly marked with the title "Perpective Transformation".  I first define the source and destination points.  I them in the following manner:

```python

#  Source points

left_bottom = [205,img_size[1]]
right_bottom = [img_size[0]-140, img_size[1]]
left_top = [img_size[0]//2-60 ,460]
right_top = [img_size[0]//2+70,460]
src = np.float32([left_bottom,right_bottom, right_top, left_top])

des_left_offset = 320
des_right_offset = 320
des_left_bottom = [des_left_offset, img_size[1]]
des_right_bottom = [img_size[0]-des_right_offset, img_size[1]]
des_left_top = [des_left_offset, 0]
des_right_top = [img_size[0]-des_right_offset, 0]
des = np.float32([des_left_bottom,des_right_bottom, des_right_top, des_left_top]) 

```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 205, 720      | 320, 720        | 
| 1140, 720      | 960, 720      |
| 710, 460     | 960, 0      |
| 580, 460      | 320, 0        |

I verified that my perspective transform was working as expected by drawing the source and destination points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image6]
![alt text][image7]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I used two approaches to identified lane-line pixels. 

The first approach is called sliding window search.  I first identified the starting points for the left and right lines with a histogram of the bottom half of the image. Then I divided the the image equally into 9 sections in vertical direction. For the first section, I searched pixels within a rectangular window centering the starting points. When the number of pixels went over one limit, I recentered the window for the next window search. I repeated these steps until reaching the top of the image. Then, I fitted a second order polynomial With the identified pixels  for each line. An example is shown below.
![alt text][image8]


The second approach is called searching around detected lines in the previous frame. Because the line curvature does not change abruptly from frame to frame, it is reasonable to do this targeted search. The following image shows the search region, found pixels, and fitted lines.
![alt text][image9]


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The codes for the calculations of radius of curvature and vehicle center offset are clearly marked with the title " Compute the radius of curvature and center offset". 

The units of the fitted lines are pixels. I first converted the fitted lines in pixels to those in meters with the following codes

```
# Define conversions in x and y from pixels space to meters
ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension
left_fit_cr = np.array([left_fit[0]*xm_per_pix/(ym_per_pix**2),left_fit[1]*xm_per_pix/ym_per_pix,left_fit[2]*xm_per_pix ]) 
right_fit_cr = np.array([right_fit[0]*xm_per_pix/(ym_per_pix**2),right_fit[1]*xm_per_pix/ym_per_pix,right_fit[2]*xm_per_pix ])       
```

After the transfer of units, I evaluated the curvature of radius at the bottom of the image for both lines with the codes

```
# Generate x and y values for plotting
ploty = np.linspace(0, binary_warped.shape[0]-1, binary_warped.shape[0])
# Define y-value where we want radius of curvature
# We'll choose the maximum y-value, corresponding to the bottom of the image
y_eval = np.max(ploty)
##### Implement the calculation of R_curve (radius of curvature) #####
left_curverad = ((1+(2*left_fit_cr[0]*y_eval*ym_per_pix+left_fit_cr[1])**2 )**(3/2))/(2*left_fit_cr[0])  ## Implement the calculation of the left line 
right_curverad =  ((1+(2*right_fit_cr[0]*y_eval*ym_per_pix+right_fit_cr[1])**2 )**(3/2))/(2*right_fit_cr[0])   ## Implement the calculation of the right line 

```
I took the average of the curvature of radius for the left line and that for the right line as the final output. 

To calculate the center offset, I first got the 'x' position for both left and right fitted lines at the bottom of the image. Then I calcualted the lane center as an average of these two values. The car center is assumed to be located at the image center. The position of the vehicle with respect to lane center is equal to the absolute value of 'lane center postion - car center position'.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The codes for this step can be found under the title "Inverse Transform".  Here is an example of my result on a test image with curvature of radius and center offset provided.

![alt text][image10]

---

### Pipeline (video)

At the end, I finalized a pipeline that can be used to process videos.

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

I also tried on the challenge video, but the challenge video output is not as good as the project video output. Here's a [link to my challenge video result](./challenge_video_output.mp4)

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

One issue I met is that when I applied the color threshold, I mistakenly used  R chanel in  150 < R <255 and G chanel in 150 < G <255. I did not include 255, which cut off a lot of line pixels.  It still worked for the test images, but did not work for the project video. I spent days in debugging until I found the mistake.


My pipeline is likely to fail under strong lighting conditions and heavy shadows. In the project video output, just in a short moment, the fitted line deviates from the true line under strong lighting conditions. The deviation is not large, but still obvious.  If the car is keeping running under strong lighting conditions, the deviation may become large and the pipeline is likely to fail. For the challenge video, the pipeline is not good to fit the lines when the car is under the bridge.  


One way to make my pipelines more robust is to dynamically adjusting the thresholding parameters. It is possible to get one set of parameters that work for strong lighting condtions, and another set of parameters that work for heavy shadows. But we probably are not able to get one set of parameters that work for all road conditions and weather conditions. If the pipeline is able to accounts for the road conditions and weather conditions, and adjusts the thresholding parameters, it will be much more robust. 



