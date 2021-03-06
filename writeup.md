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
	
	
```python
# 9*6 boards
objp = np.zeros((6*9,3), np.float32)
objp[:,:2] = np.mgrid[0:9, 0:6].T.reshape(-1,2)

# Arrays to store object points and image points from all the images.
objpoints = [] # 3d points in real world space
imgpoints = [] # 2d points in image plane.

# Make a list of calibration images
images = glob.glob('./camera_cal/calib*.jpg')
print("Calibration images found: " ,len(images))
# Step through the list and search for chessboard corners
for idx, fname in enumerate(images):
    img = cv2.imread(fname)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

    # Find the chessboard corners
    ret, corners = cv2.findChessboardCorners(gray, (9,6), None)
    # If found, add object points, image points
    if ret == True:
        objpoints.append(objp)
        imgpoints.append(corners)

        # Draw and display the corners
        cv2.drawChessboardCorners(img, (9,6), corners, ret)
        '''write_name = 'corners_found'+str(idx+1)+'.jpg'
        cv2.imwrite(write_name, img)'''
        cv2.imshow('img', img)
        cv2.waitKey(500)
    else:
        print("False image calib return: ", idx+1)

cv2.destroyAllWindows()

#calibrate the camera and write the files
'''img = cv2.imread('./camera_cal/calibration1.jpg')
img_size = (img.shape[1], img.shape[0])


ret, mtx, dist, rvecs, tvecs = cv2.calibrateCamera(objpoints, imgpoints, img_size,None,None)
# Save the camera calibration result for later use (we won't worry about rvecs / tvecs)
dist_pickle = {}
dist_pickle["mtx"] = mtx
dist_pickle["dist"] = dist
pickle.dump( dist_pickle, open( "calibration.p", "wb" ) )'''

print("done")
```

    Calibration images found:  20
    False image calib return:  1
    False image calib return:  15
    False image calib return:  16
    done
    

# Distortion Correction

### Example of the undistortion results


```python
#example road image
test_img = mpimg.imread('./test_images/test2.jpg')
#example distorted checkboard image
test_check = mpimg.imread('./camera_cal/calibration1.jpg')

#undistort
test_undist = cv2.undistort(test_img, mtx, dst, None, mtx)
test_check = cv2.undistort(test_check, mtx, dst, None, mtx)

#write the example undistorted checkboard
cv2.imwrite('./camera_cal/test_undist.jpg',test_check)

# Visualize undistortion
f, (ax1, ax2) = plt.subplots(1, 2, figsize=(20,10))
ax1.imshow(test_img)
ax1.set_title('Original Image', fontsize=30)
ax2.imshow(test_undist)
ax2.set_title('Undistorted Image', fontsize=30)
```




    <matplotlib.text.Text at 0x2176eca6518>


	
![alt text][image1]
	
The image above will then be wrapped into birds eye view only showing a section of the road in front of the car. 
	
```python
def warp(img):
    img_size = (img.shape[1], img.shape[0])
    w = img.shape[1]
    h = img.shape[0]
    #tr,br,bl,tl
    src = np.float32([(580, 460), (205, 720), (1110, 720), (703, 460)])
    dst = np.float32([(320, 0), (320, 720), (960, 720), (960, 0)])
    
    M = cv2.getPerspectiveTransform(src,dst)
    Minv = cv2.getPerspectiveTransform(dst,src)
    warped = cv2.warpPerspective(img,M,img_size,flags=cv2.INTER_LINEAR)
    return warped, Minv
```


```python
test_warp,Minv = warp(test_img)
plt.imshow(test_warp)
```




    <matplotlib.image.AxesImage at 0x2176edde828>
	
![alt text][image2]
	
Now we have to process these images 
we pass it through a color filter and perform the sobel operator to get the binary image with the lane lines(yellow and white) detected.

```python
def pipeline(img, s_thresh=(150,255), sx_thresh=(45,110)):
    img,minv = warp(img)
    gray = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
    hls = cv2.cvtColor(img, cv2.COLOR_RGB2HLS)
        
    #values to define color white boundaries in hsl
    lower_white = np.array([0,200,0], dtype = "uint8")
    upper_white = np.array([255,255,255], dtype = "uint8")

    #values to define yellow boundaries in hsl
    lower_yellow = np.array([10,0,100], dtype = "uint8")
    upper_yellow = np.array([40,200,255], dtype = "uint8")

    #create NumPy arrays from the boundaries and create the masks
    lowerW = np.array(lower_white, dtype = "uint8")
    upperW = np.array(upper_white, dtype = "uint8")
    mask_white = cv2.inRange(hls,lowerW,upperW)
    
    lowerY = np.array(lower_yellow, dtype = "uint8")
    upperY = np.array(upper_yellow, dtype = "uint8")
    mask_yellow = cv2.inRange(hls,lowerY,upperY)
    
    #bitwise OR the white and yellow masks
    mask = cv2.bitwise_or(mask_white,mask_yellow)
    
    #bitwise AND the grayscale image and the mask
    output = cv2.bitwise_and(gray,gray,mask=mask)
    
    # Sobel x
    sobelx = cv2.Sobel(output, cv2.CV_64F, 1, 0) # Take the derivative in x
    abs_sobelx = np.absolute(sobelx) # Absolute x derivative to accentuate lines away from horizontal
    scaled_sobel = np.uint8(255*abs_sobelx/np.max(abs_sobelx))
    
    # Threshold x gradient
    sxbinary = np.zeros_like(scaled_sobel)
    sxbinary[(scaled_sobel >= sx_thresh[0]) & (scaled_sobel <= sx_thresh[1])] = 1
    
    # Threshold color channel
    s_binary = np.zeros_like(output)
    s_binary[(output >= s_thresh[0]) & (output <= s_thresh[1])] = 1
    # Stack each channel
    # Note color_binary[:, :, 0] is all 0s, effectively an all black image. It might
    # be beneficial to replace this channel with something else.
    color_binary = np.zeros_like(sxbinary)
    color_binary[(s_binary == 1) | (sxbinary == 1)] = 1
    
    return color_binary, minv
```


```python
def show_masks(test_warp):
    img_pipe = test_warp
    gray = cv2.cvtColor(img_pipe, cv2.COLOR_RGB2GRAY)
    hls = cv2.cvtColor(img_pipe, cv2.COLOR_RGB2HLS)

    #values to define color white boundaries in hsl
    lower_white = np.array([0,200,0], dtype = "uint8")
    upper_white = np.array([255,255,255], dtype = "uint8")

    #values to define yellow boundaries in hsl
    lower_yellow = np.array([10,0,100], dtype = "uint8")
    upper_yellow = np.array([40,200,255], dtype = "uint8")

    #create NumPy arrays from the boundaries and create the masks
    lowerW = np.array(lower_white, dtype = "uint8")
    upperW = np.array(upper_white, dtype = "uint8")
    mask_white = cv2.inRange(hls,lowerW,upperW)

    lowerY = np.array(lower_yellow, dtype = "uint8")
    upperY = np.array(upper_yellow, dtype = "uint8")
    mask_yellow = cv2.inRange(hls,lowerY,upperY)

    #bitwise OR the white and yellow masks
    mask = cv2.bitwise_or(mask_white,mask_yellow)

    #bitwise AND the grayscale image and the mask
    test_output = cv2.bitwise_and(gray,gray,mask=mask)

    test_pipe, Minv = pipeline(test_img)
    return mask,test_output, test_pipe
mask,test_output, test_pipe = show_masks(test_warp)
fig, axs = plt.subplots(1,3, figsize=(10, 20))
fig.subplots_adjust(hspace = .2, wspace=.1)
axs = axs.ravel()
axs[0].imshow(test_warp)
axs[0].axis('off')
axs[1].imshow(test_output)
axs[1].axis('off')
axs[2].imshow(test_pipe)
axs[2].axis('off')
#just for testing
test_pipe, Minv2 = pipeline(test_img)
```

then we go through the pipeline 

```python
images = glob.glob('./test_images/*.jpg')
                                          
# Set up plot
fig, axs = plt.subplots(len(images),2, figsize=(10, 20))
fig.subplots_adjust(hspace = .2, wspace=.001)
axs = axs.ravel()
                  
i = 0
for image in images:
    img = cv2.imread(image)
    img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    img_bin, Minv = pipeline(img)
    axs[i].imshow(img)
    axs[i].axis('off')
    i += 1
    axs[i].imshow(img_bin, cmap='gray')
    axs[i].axis('off')
    i += 1

print("done")
```

    done
	 
	 and we get
 ![alt text][image3]
	 
* Next, now we need to pass the image through a sliding window function or "predict_lanes" function. 
    The sliding window performs a search by sliding a box over the image to identify the peaks and clusters of detected lanes from a histogram of the image and creates a set of variables to carry the cluster information for both left and right lanes.
	
	```python
def sliding_window(binary_warped):
    histogram = np.sum(binary_warped[binary_warped.shape[0]//2:,:], axis=0)
    # Take a histogram of the bottom half of the image
    #histogram = np.sum(binary_warped[binary_warped.shape[0]/2:,:], axis=0)
    # Create an output image to draw on and  visualize the result
    out_img = np.dstack((binary_warped, binary_warped, binary_warped))*255
    # Find the peak of the left and right halves of the histogram
    # These will be the starting point for the left and right lines
    midpoint = np.int(histogram.shape[0]/2)
    leftx_base = np.argmax(histogram[:midpoint])
    rightx_base = np.argmax(histogram[midpoint:]) + midpoint

    # Choose the number of sliding windows
    nwindows = 10
    # Set height of windows
    window_height = np.int(binary_warped.shape[0]/nwindows)
    # Identify the x and y positions of all nonzero pixels in the image
    nonzero = binary_warped.nonzero()
    nonzeroy = np.array(nonzero[0])
    nonzerox = np.array(nonzero[1])
    # Current positions to be updated for each window
    leftx_current = leftx_base
    rightx_current = rightx_base
    # Set the width of the windows +/- margin
    margin = 85
    # Set minimum number of pixels found to recenter window
    minpix = 40
    # Create empty lists to receive left and right lane pixel indices
    left_lane_inds = []
    right_lane_inds = []

    # Step through the windows one by one
    for window in range(nwindows):
        # Identify window boundaries in x and y (and right and left)
        win_y_low = binary_warped.shape[0] - (window+1)*window_height
        win_y_high = binary_warped.shape[0] - window*window_height
        win_xleft_low = leftx_current - margin
        win_xleft_high = leftx_current + margin
        win_xright_low = rightx_current - margin
        win_xright_high = rightx_current + margin
        # Draw the windows on the visualization image
        cv2.rectangle(out_img,(win_xleft_low,win_y_low),(win_xleft_high,win_y_high),
        (0,255,0), 3) 
        cv2.rectangle(out_img,(win_xright_low,win_y_low),(win_xright_high,win_y_high),
        (0,255,0), 3) 
        # Identify the nonzero pixels in x and y within the window
        good_left_inds = ((nonzeroy >= win_y_low) & (nonzeroy < win_y_high) & 
        (nonzerox >= win_xleft_low) &  (nonzerox < win_xleft_high)).nonzero()[0]
        good_right_inds = ((nonzeroy >= win_y_low) & (nonzeroy < win_y_high) & 
        (nonzerox >= win_xright_low) &  (nonzerox < win_xright_high)).nonzero()[0]
        # Append these indices to the lists
        left_lane_inds.append(good_left_inds)
        right_lane_inds.append(good_right_inds)
        # If you found > minpix pixels, recenter next window on their mean position
        if len(good_left_inds) > minpix:
            leftx_current = np.int(np.mean(nonzerox[good_left_inds]))
        if len(good_right_inds) > minpix:        
            rightx_current = np.int(np.mean(nonzerox[good_right_inds]))

    # Concatenate the arrays of indices
    left_lane_inds = np.concatenate(left_lane_inds)
    right_lane_inds = np.concatenate(right_lane_inds)

    # Extract left and right line pixel positions
    leftx = nonzerox[left_lane_inds]
    lefty = nonzeroy[left_lane_inds] 
    rightx = nonzerox[right_lane_inds]
    righty = nonzeroy[right_lane_inds]
    
    left_fit, right_fit = (None, None)
    # Fit a second order polynomial to each
    if len(leftx) != 0:
        left_fit = np.polyfit(lefty, leftx, 2)
    if len(rightx) != 0:
        right_fit = np.polyfit(righty, rightx, 2)
    
    # Generate x and y values for plotting


    out_img[nonzeroy[left_lane_inds], nonzerox[left_lane_inds]] = [255, 0, 0]
    out_img[nonzeroy[right_lane_inds], nonzerox[right_lane_inds]] = [0, 0, 255]

    return out_img, left_fit, right_fit, left_lane_inds, right_lane_inds

```

### Example sliding window and histogram result


```python
histogram = np.sum(test_pipe[test_pipe.shape[0]//2:,:], axis=0)
plt.plot(histogram)

def draw_sliding(out_img, left_fit, right_fit, left_lane_inds, right_lane_inds):
    
    plt.figure()
    # Generate x and y values for plotting
    ploty = np.linspace(0, test_pipe.shape[0]-1, test_pipe.shape[0] )
    left_fitx = left_fit[0]*ploty**2 + left_fit[1]*ploty + left_fit[2]
    right_fitx = right_fit[0]*ploty**2 + right_fit[1]*ploty + right_fit[2]
    nonzero = test_pipe.nonzero()
    nonzeroy = np.array(nonzero[0])
    nonzerox = np.array(nonzero[1])
    out_img[nonzeroy[left_lane_inds], nonzerox[left_lane_inds]] = [255, 0, 0]
    out_img[nonzeroy[right_lane_inds], nonzerox[right_lane_inds]] = [0, 0, 255]
    #plt.imshow(out_img)
    return out_img, left_fitx, right_fitx, ploty

out_img, left_fit, right_fit, left_lane_inds, right_lane_inds = sliding_window(test_pipe)

dsw_result, left_fitx, right_fitx, ploty = draw_sliding(out_img, left_fit, right_fit, left_lane_inds, right_lane_inds)
plt.imshow(dsw_result)
plt.plot(left_fitx, ploty, color='yellow')
plt.plot(right_fitx, ploty, color='yellow')
plt.xlim(0, 1280)
plt.ylim(720, 0)
```




    (720, 0)

	
	![alt text][image4]          	![alt text][image5]
	
 	* The "predict_lanes" will be used to speed up the process by using information from the sliding window to draw out the lanes.

	```python
def predict_lanes(binary_warped, left_fit, right_fit):
    # Assume you now have a new warped binary image 
    # from the next frame of video (also called "binary_warped")
    # It's now much easier to find line pixels!
    nonzero = binary_warped.nonzero()
    nonzeroy = np.array(nonzero[0])
    nonzerox = np.array(nonzero[1])
    margin = 85
    left_lane_inds = ((nonzerox > (left_fit[0]*(nonzeroy**2) + left_fit[1]*nonzeroy + 
    left_fit[2] - margin)) & (nonzerox < (left_fit[0]*(nonzeroy**2) + 
    left_fit[1]*nonzeroy + left_fit[2] + margin))) 

    right_lane_inds = ((nonzerox > (right_fit[0]*(nonzeroy**2) + right_fit[1]*nonzeroy + 
    right_fit[2] - margin)) & (nonzerox < (right_fit[0]*(nonzeroy**2) + 
    right_fit[1]*nonzeroy + right_fit[2] + margin)))  

    # Again, extract left and right line pixel positions
    leftx = nonzerox[left_lane_inds]
    lefty = nonzeroy[left_lane_inds] 
    rightx = nonzerox[right_lane_inds]
    righty = nonzeroy[right_lane_inds]
    # Fit a second order polynomial to each
    left_fit_new, right_fit_new = (None, None)
    if len(leftx) != 0:
        # Fit a second order polynomial to each
        left_fit_new = np.polyfit(lefty, leftx, 2)
    if len(rightx) != 0:
        right_fit_new = np.polyfit(righty, rightx, 2)
    
    return left_fit_new, right_fit_new, left_lane_inds, right_lane_inds

```

### Visualize output of predict lanes function


```python
def draw_predict(left_fit, right_fit, left_fit2, right_fit2, left_lane_inds2, right_lane_inds2):
    margin = 85
    # Generate x and y values for plotting
    ploty = np.linspace(0, test_pipe.shape[0]-1, test_pipe.shape[0] )
    left_fitx = left_fit[0]*ploty**2 + left_fit[1]*ploty + left_fit[2]
    right_fitx = right_fit[0]*ploty**2 + right_fit[1]*ploty + right_fit[2]
    left_fitx2 = left_fit2[0]*ploty**2 + left_fit2[1]*ploty + left_fit2[2]
    right_fitx2 = right_fit2[0]*ploty**2 + right_fit2[1]*ploty + right_fit2[2]

    # Create an image to draw on and an image to show the selection window
    out_img = np.uint8(np.dstack((test_pipe, test_pipe, test_pipe))*255)
    window_img = np.zeros_like(out_img)

    # Color in left and right line pixels
    nonzero = test_pipe.nonzero()
    nonzeroy = np.array(nonzero[0])
    nonzerox = np.array(nonzero[1])
    out_img[nonzeroy[left_lane_inds2], nonzerox[left_lane_inds2]] = [255, 0, 0]
    out_img[nonzeroy[right_lane_inds2], nonzerox[right_lane_inds2]] = [0, 0, 255]

    # Generate a polygon to illustrate the search window area (OLD FIT)
    # And recast the x and y points into usable format for cv2.fillPoly()
    left_line_window1 = np.array([np.transpose(np.vstack([left_fitx-margin, ploty]))])
    left_line_window2 = np.array([np.flipud(np.transpose(np.vstack([left_fitx+margin, ploty])))])
    left_line_pts = np.hstack((left_line_window1, left_line_window2))
    right_line_window1 = np.array([np.transpose(np.vstack([right_fitx-margin, ploty]))])
    right_line_window2 = np.array([np.flipud(np.transpose(np.vstack([right_fitx+margin, ploty])))])
    right_line_pts = np.hstack((right_line_window1, right_line_window2))

    # Draw the lane onto the warped blank image
    cv2.fillPoly(window_img, np.int_([left_line_pts]), (0,255, 0))
    cv2.fillPoly(window_img, np.int_([right_line_pts]), (0,255, 0))
    result = cv2.addWeighted(out_img, 1, window_img, 0.3, 0)
    #plt.imshow(result)
    #plt.plot(left_fitx2, ploty, color='yellow')
    #plt.plot(right_fitx2, ploty, color='yellow')
    #plt.xlim(0, 1280)
    #plt.ylim(720, 0)
    return result, left_fitx2, right_fitx2, ploty

left_fit2, right_fit2, left_lane_inds2, right_lane_inds2 = predict_lanes(test_pipe, left_fit, right_fit)

dpl_result, left_fitx2, right_fitx2, ploty = draw_predict(left_fit, right_fit, left_fit2, right_fit2, left_lane_inds2, right_lane_inds2)
plt.imshow(dpl_result)
plt.plot(left_fitx2, ploty, color='yellow')
plt.plot(right_fitx2, ploty, color='yellow')
plt.xlim(0, 1280)
plt.ylim(720, 0)
```




    (720, 0)
	
	![alt text][image6]

	* The output cluster information is then used to calculate the curvature and determine the distance the car is away from the center of the lane(negative being to the left, positive to the right).

	The information is then compiled into one image and the lane detected + the curvature and distance information is draw onto the final result and compiled into a video out(see project_video_output.mp4)
		
	# Calculate the curvature and distance from center


```python
def calc_curv_rad_and_center_dist(bin_img, l_fit, r_fit, l_lane_inds, r_lane_inds):
    # Define conversions in x and y
    ym_per_pix = 30/720
    xm_per_pix = 3.7/700 
    left_curverad, right_curverad, center_dist = (0, 0, 0)
    # Define y-value where we want radius of curvature one less than height of image
    h = bin_img.shape[0]
    ploty = np.linspace(0, h-1, h)
    y_eval = np.max(ploty)
  
    #x and y positions of all nonzero pixels in the image
    nonzero = bin_img.nonzero()
    nonzeroy = np.array(nonzero[0])
    nonzerox = np.array(nonzero[1])
    #Left and right line pixel positions
    leftx = nonzerox[l_lane_inds]
    lefty = nonzeroy[l_lane_inds] 
    rightx = nonzerox[r_lane_inds]
    righty = nonzeroy[r_lane_inds]
    
    if len(leftx) != 0 and len(rightx) != 0:
        # Fit new polynomials to x,y in world space
        left_fit_cr = np.polyfit(lefty*ym_per_pix, leftx*xm_per_pix, 2)
        right_fit_cr = np.polyfit(righty*ym_per_pix, rightx*xm_per_pix, 2)
        # Calculate the new radii of curvature
        left_curverad = ((1 + (2*left_fit_cr[0]*y_eval*ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
        right_curverad = ((1 + (2*right_fit_cr[0]*y_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])
        # Now our radius of curvature is in meters
    
    # Distance from center is image x midpoint - mean of l_fit and r_fit intercepts 
    if r_fit is not None and l_fit is not None:
        car_position = bin_img.shape[1]/2
        l_fit_x_int = l_fit[0]*h**2 + l_fit[1]*h + l_fit[2]
        r_fit_x_int = r_fit[0]*h**2 + r_fit[1]*h + r_fit[2]
        lane_center_position = (r_fit_x_int + l_fit_x_int) /2
        center_dist = (car_position - lane_center_position) * xm_per_pix
    return left_curverad, right_curverad, center_dist
```

### Example output of the curvature and distance


```python
#rad_l, rad_r, d_center = calc_curv_rad_and_center_dist(warped_res, left_fit, right_fit, left_lane_inds, right_lane_inds)
rad_l, rad_r, d_center = calc_curv_rad_and_center_dist(test_pipe, left_fit, right_fit, left_lane_inds, right_lane_inds)

print('Radius of curvature for example:', rad_l, 'm,', rad_r, 'm')
print('Distance from lane center for example:', d_center, 'm')
```

    Radius of curvature for example: 710.450316788 m, 1399.19706339 m
    Distance from lane center for example: -0.351957023878 m
    

# Draw the detected lanes over the image*


```python
def draw_result(original_img, binary_img, l_fit, r_fit, Minv):
    new_img = np.copy(original_img)
    if l_fit is None or r_fit is None:
        return original_img
    # Create an image to draw the lines on
    warp_zero = np.zeros_like(binary_img).astype(np.uint8)
    color_warp = np.dstack((warp_zero, warp_zero, warp_zero))
    
    h,w = binary_img.shape
    ploty = np.linspace(0, h-1, num=h)# to cover same y-range as image
    left_fitx = l_fit[0]*ploty**2 + l_fit[1]*ploty + l_fit[2]
    right_fitx = r_fit[0]*ploty**2 + r_fit[1]*ploty + r_fit[2]

    # Recast the x and y points into usable format for cv2.fillPoly()
    pts_left = np.array([np.transpose(np.vstack([left_fitx, ploty]))])
    pts_right = np.array([np.flipud(np.transpose(np.vstack([right_fitx, ploty])))])
    pts = np.hstack((pts_left, pts_right))

    # Draw the lane onto the warped blank image
    cv2.fillPoly(color_warp, np.int_([pts]), (0,160, 255))
    cv2.polylines(color_warp, np.int32([pts_left]), isClosed=False, color=(255,0,0), thickness=25)
    cv2.polylines(color_warp, np.int32([pts_right]), isClosed=False, color=(0,255,0), thickness=25)

    # Warp the blank back to original image space using inverse perspective matrix (Minv)
    newwarp = cv2.warpPerspective(color_warp, Minv, (w, h)) 
    # Combine the result with the original image
    result = cv2.addWeighted(new_img, 1, newwarp, 0.5, 0)
    return result
```

### Example of drawn lanes


```python
exampleImg_out1 = draw_result(test_img, test_pipe, left_fit, right_fit, Minv2)
plt.imshow(exampleImg_out1)
```




    <matplotlib.image.AxesImage at 0x217729c7ac8>
	
	![alt text][image7]	

# Discussion 

Future improvements to the work flow would include a change in the distance(make it shorter) that the initial warp looks at, creating a smaller image that would work better for turns and an update to the color masking. The current color masking identifies the yellow and white lines through HLS color space which includes some light shadows but is unusable in the exteme situations. Exploring more color spaces and different thresholds could lead to a much better pipeline and lane detection results.



