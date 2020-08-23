## Writeup 

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

[image1]: ./output_images/undist_image.jpg "Undistorted"
[image2]: ./output_images/unwarp_image.jpg "Road Transformed"
[image3]: ./output_images/thresh_image.jpg "Binary Image"
[image4]: ./output_images/uncalib_image.jpg "Distorted"
[image5]: ./output_images/fit_image.jpg "Fit Visual"
[image6]: ./output_images/processed_image.jpg "Output"
[image7]: ./output_images/corrected_image.jpg "Correct"

[video1]: ./Processed_project_video.mp4 "Video"


---

### Writeup / README

The code is in the IPython notebook located in `./P2.ipynb`.

### Camera Calibration

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![Distorted Image][image4]
>Distorted Image

![Undistorted Image][image1]
>Calibtared Image

### Pipeline (single images)

#### 1. Distortion-corrected image.

Using the `mtx` and `dist` from the `cv2.calibrateCamera()`, the distored image can be corrested.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![Corrected Image][image7]
>Distortion-Corrected Image

#### 2. Binary image result.

For generating the binary image result, I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at the third code cell in `P2.ipynb`). For color threshold, I converted image to hls first, then selected the s_channel and l_channel. For gradient threshold, I applied a Sobel operator to X direction. Here's an example of my output for this step. 

![Binary Image][image3]
>Binary Output

#### 3. Perspective transform.

The code for my perspective transform includes a function called `unwarp()`, which appears in fifth code cell in the file `P2.ipynb`.  The `unwarp()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```
    src = np.float32(
        [[280, img.shape[0]], #bottom left
        [1120, img.shape[0]], #bottom right
        [590, 460], #top left
        [720, 460] #top right
        ])
    dst = np.float32(
        [[340, img.shape[0]],
        [1060, img.shape[0]], 
        [340, 0],
        [1060, 0]
        ])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 280, 720      | 340, 720      | 
| 1120, 720     | 1060, 720     |
| 590, 460      | 340, 0        |
| 720, 460      | 1060, 0       |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![Transformed Image][image2]
>Bird-Eye View

#### 4. Lane fitting.

First, I used a sliding windows mechanism with 15 windows to locate the peaks of histogram. The windows would be sliding across the image and locate the two lane lines. Then the 2nd order polynomial could be fit with the help of these windows.

Then, I used another function called `search_around_poly()`. This function utlized the result from previous sliding windows mechanism and search the polynomial fit around a margin of 100. And this function would create another 2nd order polynomial fit.

Next, I filled the region between the 2 polynominal lines with green color.

An example is shown below:

![Fit Visual][image5]
>Lane Fitted Image

#### 5. Radius of curvature of the lane & the position of the vehicle with respect to center.

First, I set 2 conversion relations. From real world to image, meters per pixel in x dimension is `3.7/700`. And meters per pixel in y dimension is `30/720`.

Then I converted the fitted polynominal lines to real-world scale, and calculate left curvature and right curvature separately. After taken the mean of the 2 curvatures, the lane radius could be found.

To calculate the lane center, I used the un-converted polynominal lines first, and pluged in a number to get 2 values. The lane center could be found by taken the mean of these 2 values.
`lane_center =np.mean([left_fitx[0],right_fitx[0]])`
For the vehicle center, I assumed the camera was place at the center of the car. And the image center should be the same as the vehicle center. Divided the image width by 2, we could found the vehicle center.

I did this in the eighth code cell in `P2.ipynb`.

#### 6. Final output.

With the radius of curvature calculated, next step was to do a inverse transpose of the wrapped image and overlaied the image on top of the original image. Then, the text of lane radius and vehicle position would be projected onto the overlay image.  Here is an example of my result on a test image:

![Final Output][image6]
>Processed Image

---

### Pipeline (video)

The video is called "Processed_project_video.mp4", and it contained in the submitted zip file.

---

### Discussion

From the processed video, the lane finding mechanism performed as I expected. However, there are a few problems.

1. The calculated lane curvature is wobble. Sometimes, it bounces up to a very huge number and then return back to normal. I think this is due to the image transpose. The "bird-eye view" distord the original image too much or too less to cause the radius calculation funciton malfuntioning.

2. The image wrapping function (to generate the bird-eye view) is not very robust. In the challenge video and harder challenge video, my pipeline cannot finish processing the entire video. I looked at the error, the function cannot fit a 2nd polynomial to the left lane. This might caused by the transposed image (bird-eye view image) being distored too much and the lane fitting function falied to fint a line.

3. For the pipeline, the video processing speed it very slow. I understand the sliding windows mechanism should be only used once before the `search_aorund_poly()` funciton, when processing video. I tried a few things (like adding a `__init__` function) to just run the sliding windows function once and then just the `search_around_poly()` funciton, but none of them turned out well in my pipeline. So, both functions are in my pipeline and that made the processing speed pretty slow. I would like to know how to accomplish this.

4. Due to some personal issues and limited time, I only finished the base-line of this project. And I am not really satisfacted with my finished project. After submitted before the required deadline, I will keep modifying and complete this project.
