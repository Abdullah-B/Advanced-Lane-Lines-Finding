# Advanced Lane Detection

[//]: # (Image References)

[image1]: undistorted.png "Undistorted"
[image2]: wrap.png "Fish eye Wrap"
[image3]: ResultsPipeline.png "Pipeline Results"
[image4]: Histogram.png  "Histogram"
[image5]: Slidingwindow.png "Sliding Window"
[image6]: visualize.png "visualization of predictions"
[image7]: drawingLine.png "Drawing lines"


 ##
 this project will look through a video and identify the lanes in front of the car 

   # The process goes as follows 

First it will calibrate the camera using 9 by 6 chess board(see output_images/calibration_lines folder) the matrix and distortion coefficients.
	
The camera calibration will help increase the accuracy of detecting and eliminating distortions from the  camera .
	
![alt text][image1]
	
The image above will then be wrapped into birds eye view only showing a section of the road in front of the car. 
	
![alt text][image2]
	
Now we have to process these images 
we pass it through a color filter and perform the sobel operator to get the binary image with the lane lines(yellow and white) detected.
and we get 
	 
 ![alt text][image3]
	 
* Next, now we need to pass the image through a sliding window function or "predict_lanes" function. 
    The sliding window performs a search by sliding a box over the image to identify the peaks and clusters of detected lanes from a histogram of the image and creates a set of variables to carry the cluster information for both left and right lanes.
	
	![alt text][image4]          	![alt text][image5]
	
 	* The "predict_lanes" will be used to speed up the process by using information from the sliding window to draw out the lanes.

	![alt text][image6]

	* The output cluster information is then used to calculate the curvature and determine the distance the car is away from the center of the lane(negative being to the left, positive to the right).

	The information is then compiled into one image and the lane detected + the curvature and distance information is draw onto the final result and compiled into a video out(see project_video_output.mp4)
		
	![alt text][image7]	




