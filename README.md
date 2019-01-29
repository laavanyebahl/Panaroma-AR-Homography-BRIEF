# Finding keypoints and correspondences using BRIEF
We implement a function ```locs, desc = briefLite(im)``` to extract the descriptors from the image, including:
• Compute DoG pyramid.
• Get keypoint locations.
• Compute a set of valid BRIEF descriptors.

For BRIEF, the distance metric for matching is the Hamming distance.      
```matches = briefMatch(desc1, desc2, ratio)```    
accepts an m1 × n bits stack of BRIEF descriptors from a first image and a
m2×n bits stack of BRIEF descriptors from a second image and returns a p×2 matrix of
matches, where the first column are indices into desc1 and the second column are indices
into desc2. 


We implement the  Planar Homographies function
```H2to1 = computeH(X1,X2)```
Inputs: X1 and X2 should be 2 × N matrices of corresponding (x; y)T coordinates between
two images.
Outputs: H2to1 should be a 3 × 3 matrix encoding the homography that best matches
the linear equation derived above for Equation 8 (in the least squares sense).

![2d](/results/brief.png)

**RANSAC**    
The least squares method you implemented for computing homographies is not robust to outliers. If all the correspondences are good matches, this is not a problem. But even a single false correspondence can completely throw off the homography estimation. When correspondences are determined automatically (using BRIEF feature matching for instance), some mismatches in a set of point correspondences are almost certain. RANSAC (Random Sample Consensus can be used to fit models robustly in the presence of outliers.
Below funciton uses RANSAC to compute homographies automatically between two images:    
```bestH = ransacH(matches, locs1, locs2, nIter, tol)```

• Algorithm Input Parameters: nIter is the number of iterations to run RANSAC for, tol is the tolerance value for considering a point to be an inlier.
• Outputs: bestH should be the homography model with the most inliers found during RANSAC.

# Panaroma-AR-Homography
Impementation of Panaroma and Augmented Reality (AR) using Homography.

We can use homographies to create a panorama image from multiple views of the same scene. This is possible for example when there is no camera translation between the views (e.g., only rotation about the camera center).

First, we generate panoramas using matched point correspondences between images using the BRIEF descriptor matching (implemented in /code/BRIEF.py). 

We use the perspective warping function from OpenCV:     
```warp im = cv2.warpPerspective(im, H, out size),```    
which warps image im using the homography transform H. The pixels in warp_im are sampled at coordinates in the rectangle (0; 0) to (out_size[0]-1; out_size[1]-1).

The coordinates of the pixels in the source image are taken to be (0; 0) to (im.shape[1]-1; im.shape[0]-1) and transformed according to H. 

We implement ```panoImg = imageStitching(img1, img2, H2to1)``` on two images from the Dusquesne incline. 

This function accepts two images and the output from the homography estimation function. This function:
(a) Warps img2 into img1’s reference frame using the aforementioned perspective warping function
(b) Blends img1 and warped img2 and outputs the panorama image.

The point correspondences pts are generated by the BRIEF descriptor matching.
We apply ransacH() to these correspondences to compute H2to1, which is the homography from incline R onto incline L. Then we apply this homography to incline R sing warpH().

Since the warped image will be translated to the right, we will need a larger target image.  
![1](/results/6_1.jpg)

We see the above output is clipped at the edges. We will fix this with the function:   
   
```[panoImg] = imageStitching noClip(img1, img2, H2to1)```      
that takes in the same input types and produces the same output.

To prevent clipping at the edges, we instead warp both image 1 and image 2 into a common third reference frame in which we can display both images without any clipping. Specifically, we want to find a matrix M that only does scaling and translation
such that:      
```warp im1 = cv2.warpPerspective(im1, M, out size)```     
```warp im2 = cv2.warpPerspective(im2, np.matmul(M,H2to1), out size)```   

This produces warped images in a common reference frame where all points in im1 and im2 are visible. To do this, we will only take as input either the width or height of out size and compute the other one based on the given images such that the warped
images are not squeezed or elongated in the panorama image. We take as input the width of the image (i.e., out size[0]) and  compute the correct height(i.e., out size[1]).    

The computation will be done in terms of H2to1 and the extreme points (corners) of the two images. Make sure M includes only scale (find the aspect ratio of the full-sized panorama image) and translation.  

![2](/results/q6_2_pan.jpg)

We implement a function that accepts two images as input, computes keypoints and descriptors for both the images, finds putative feature correspondences by matching BRIEF keypoint descriptors, estimates a homography using RANSAC and then warps one of the images with the homography so that they are aligned and then overlays them.
   
```im3 = generatePanorama(im1, im2)```  

We run our code on the image pair data/incline L.jpg, data/incline R.jpg.

To make the merging seemless, we find the common pixels in both the images and apply a mask of gradients to smooth out the common parts. Basically, in the corners there should be no seems. We optimize the blending of images.

Final panorama view. With homography estimated using RANSAC.

![3](/results/q6_3.jpg)



# Augmented Reality

Homographies are also really useful for Augmented Reality (AR) applications.

![4](/results/pb.jpeg)

We use the following image of the book with the following data points:-
W = 2 40 0 0: : :0 18 0 0 0 0: ::0 26 0 0 2 18:: :0 0 2 0 0 26: ::0 003 5
are the 3D planar points of the textbook (in cm);
X = 483 1704 2175 67 810 781 2217 2286
are the corresponding projected points; and
K = 2 43043 0 0: :0 3043 0 0 :72 0: :0 1196 0 1 :72 1604:0: :00 003 5
Given we have camera intrinsic parameters (K).

The relationship between 3D planar and 2D projected point can be defined through,
λnx~n = K[R;t]w~ n 

where xn and wn relate to the n-th column vector of X and W.

We have the function R,t = compute extrinsics(K, H) where K contains the intrinsic parameters and H contains the estimated homography.
The extrinsic parameters refer to the relative 3D rotation R and translation t of the camera between views. 

We have the function:
```X=project_extrinsics(K, W, R, t)```

that we apply a pinhole camera projection of any set of 3D points using a pre-deterimined set of extrinsics and intrinsics. Use this function then project the set of 3D points in the file sphere.txt (which has been fixed to have the same radius as a tennis ball - of 6.858l) onto the image of prince book.

Specifically, we attempt to place the bottom of the ball in the middle of the \o" in \Computer" text
of the book title. Use matplotlib’s plot function to display the points of the sphere in
yellow. For this question disregard self occlusion (i.e. points on the projected sphere
occluding other points). Capture the image and include it in your written response.

Result:

![5](/results/pb2.png)
