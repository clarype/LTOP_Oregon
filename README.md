### LTOP Overview

LandTrendr is a set of spectral-temporal segmentation algorithms that focuses on removing the natural spectral variations in a time series of Landsat Images. Stabling the natural variation in a time series, gives emphasis on how a landscape is evolving with time. This is useful in many pursuits as it gives information on the state of a landscape, be it growing, remaining stable, or on a decline. LandTrendr is mostly used in Google Earth Engine (GEE), an online image processing console, where it is readily available for use.  

A large obstacle in using LandTrendr in GEE, is knowing which configuration of LandTrendr parameters to use. The LandTrendr GEE function uses 9 arguments: 8 parameters that control how spectral-temporal segmentation is executed, and an annual image collection on which to assess and remove the natural variations. The original LandTrendr journal illustrates the effect and sensitivity of changing some of these values. The default parameters for the LandTrendr GEE algorithm do a satisfactory job in many circumstances, but extensive testing and time is needed to hone the parameter selection to get the best segmentation out of the LandTrendr algorithm for a given region. Thus, augmenting the Landtrendr parameter selection process would save time and standardize a method to choose parameters, but we also aim to take this augmentation a step further. 

Traditionally, LandTrendr is run over an image collection with a single LandTrendr parameter configuration and is able to remove natural variation for every pixel time series in an image. But no individual LandTrendr parameter configuration is best for all surface conditions, where forest may respond well to one configuration, but many under or over emphasize stabilization in another land class. Thus here we aim to delineate patches of spectrally similar pixels from the imagery, find what LandTrendr parameters work best for each patch group, and run LandTrendr on each patch group location with that best parameter configuration. 

In short, we take a times series of Landsat imagery, remove atmospheric contamination, and then reduce the images for each year to a single medoid composite. This gives a single image for each 
year in the time series. With the medoid time series, we use the beginning, middle and end year images in the SNIC (Simple Non-iterative Clustering) algorithm, which groups pixels that are spectrally similar to one another into clusters. Each cluster is independent of the other clusters, our next step uses K-Means to group independent clusters that are similar to one another. Then, we use the augmented parameter selection process to choose the LandTrendr parameters that work best for each cluster group. Landtrendr is then run on the medoid composite yearly time series for each cluster group location with its choosin LandTrendr parameter. This process not only augments the Landtrendr selection process but also runs LandTrendr on a patch level where each patch group has its own LandTrendr parameter.

### Steps Overview

1) Edit and run /scripts/GEEjs/01SNICPatches.js

2) Download SNIC Seed image

3) SNIC Seed image to points. (seed pixels to points)

4) Sample SNIC image with points

5) Random select a subset of point (75k)

6) upload subset of point to GEE 

7) run /scripts/GEEjs/02kMeansCluster.js with upload sample

8) Download kmeans seed and cluster imagery

9) process kmeans seek image (change no data)

10) process kmeans seek image (seed pixels to points)

11) process kmeans seek image (sample nodata image with sample points)

12) run randomDistinctSampleOfKmeansClusterIDs.py on sample points 

13) upload output to GEE

14) run abstractSampler

15) download table

16) run csv_to_abstract_images.py

17) upload shp file and tiffs to GEE. Make tiffs into image collection

18) Edit and run 04abstractImager.js in GEE

19) download table



21) upload table to GEE 

22) edit and run Lt-Optimum-Image-Printer... 



### LTOP WorkFlow (Step by Step) 

[GEE link](https://code.earthengine.google.com/https://code.earthengine.google.com/?accept_repo=users/emaprlab/SERVIR) open with Emapr Account for dependencies 



------



![img](https://lh4.googleusercontent.com/qpYv4_Q9InR0_LBzk1vdtIWhfLmMRNwZ840DSv6h0CzETzPjd2n6pgQP24eiHFQLfTKp3Tr17yLoqwdRfPeNb_YyktC60kTGnQulL7UwiLoQit-OyJJ3H_vI25-GE06J20ab_YeO=s0)

#### Run GEE [script](https://code.earthengine.google.com/bc7d5c6e42f2d738a9fe140c316adea0) to generate SNIC images (01SNICPatches.js) (cambodia 15min )

1. Open script in GEE console 
2. Make sure you all needed dependances (emapr GEE account has all dependances) 
3. Review in script parameters. Lines 35-39, lines 47-49 (SNIC years), lines 83,84 (SNIC)
4. Run script (01SNICPatches)
5. Run tasks

#### Getting data  from the Google drive to Islay

	1. Open terminal on Islay in a VNC

	2. Activate conda environment “py35”

		conda activate py35

	1. This script bring data from the google drive to Islay 

		/vol/v1/proj/SERVIR/v2_revision/LandTrendrOptimization/GEEjs/1_get_chunks_from_gdrive.py

	1. Run script 

		python /vol/v1/proj/SERVIR/v2_revision/LandTrendrOptimization/GEEjs/1_get_chunks_from_gdrive.py CambodiaSNIC_v6 /vol/v1/proj/SERVIR/v2_revision/rasters/SNIC_imagery/gee/


	1. Check data at download destination. 

		/vol/v1/proj/LTOP_Oregon/rasters/01_SNIC/



------


#### Merge image chunks into two vrt 

	1. Activate conda environment

		a) conda activate gdal37

	2. Build VRT SNIC seed image 

		a) make text file of file path is folder (only tiffs in the folder)
	
			ls -d "$PWD"/* > listOfTiffs.txt

		b) build vrt raster with text file 

			gdalbuildvrt snic_seed.vrt -input_file_list listOfTiffs.txt

	3. Build VRT SNIC image 

		a) make text file of file path is folder (only tiffs in the folder)

			ls -d "$PWD"/* > listOfTiffs.txt

		b) build vrt raster with text file 

			gdalbuildvrt snic_image.vrt -input_file_list listOfTiffs.txt


	4. Inspect and process the merged imagery.

		a) Data location:

			/vol/v1/proj/LTOP_Oregon/rasters/01_SNIC/snic_seed.vrt

			/vol/v1/proj/LTOP_Oregon/rasters/01_SNIC/snic_image.vrt			


#### Raster calc clipped seed image to keep only seed pixels (QGIS)

	1. Raster calculation (("seed_band">0)*"seed_band") / (("seed_band">0)*1 + ("seed_band"<=0)*0)

	2. Input:

		/vol/v1/proj/LTOP_Oregon/rasters/01_SNIC/snic_seed.vrt

	3. Output: 	

		/vol/v1/proj/LTOP_Oregon/rasters/01_SNIC/snic_seed_pixels.tif

	Note: 
		This raster calculation change the 0 pixel values to no data in Q-gis. However, this also 
		changes a seed id pixel to no data aswell. But one out of hundreds of millions pixels is 
		inconsequential.



#### **Change the** ***raster calc clipped seed image*** **to a vector of points. Each point corresponds to a seed pixel. (QGIS)**

	1. Qgis tool - Raster pixels to points 
  	 

	2. Input

		/vol/v1/proj/LTOP_Oregon/rasters/01_SNIC/snic_seed_pixels.tif

	3. Output

		/vol/v1/proj/LTOP_Oregon/vectors/01_SNIC/01_snic_seed_pixel_points/01_snic_seed_pixel_points.shp


#### **Sample ALL points from above image with shp file (QGIS - Sample Raster Values ~3362.35 secs)**

	1. Input point layer

		/vol/v1/proj/LTOP_Oregon/vectors/01_SNIC/01_snic_seed_pixel_points/01_snic_seed_pixel_points.shp

	2. Raster layer 

		/vol/v1/proj/LTOP_Oregon/rasters/01_SNIC/snic_seed.vrt

	3. Output column prefix

		seed_

	4. Output location 

		/vol/v1/proj/LTOP_Oregon/vectors/01_SNIC/02_snic_seed_pixel_points_attributted/02_snic_seed_pixel_points_attributted.shp


#### **Randomly select a subset of points 75k (QGIS - Random selection within subsets)**

	1. Input

		/vol/v1/proj/LTOP_Oregon/vectors/01_SNIC/02_snic_seed_pixel_points_attributted/02_snic_seed_pixel_points_attributted.shp

	2. Number of selection 

		75000 

		Note: the value above is arbitrary

	3. Save selected features as:

 		/vol/v1/proj/LTOP_Oregon/vectors/01_SNIC/03_snic_seed_pixel_points_attributted_random_subset_75k/03_snic_seed_pixel_points_attributted_random_subset_75k.shp


#### **Upload sample to gee** 

	1. Zip shape file 

		zip -r 03_snic_seed_pixel_points_attributted_random_subset_75k.zip 03_snic_seed_pixel_points_attributted_random_subset_75k/ 

	2. Up load to GEE as asset

	3. GEE Asset location 

		users/emaprlab/03_snic_seed_pixel_points_attributted_random_subset_75k




#### Kmeans of SNIC

	1. Run GEE Kmeans on SNIC image with training data from 03_snic_seed_pixel_points_attributted_random_subset_75k.shp

		5000 clusters

	2. Run GEE Kmeans on SNIC seed image with training data from 03_snic_seed_pixel_points_attributted_random_subset_75k.shp

		5000 clusters



#### **Export KMeans image to islay** 

	1. Run script 1_get_chunks_from_gdrive.py

		(py35) python 1_get_chunks_from_gdrive.py LTOP_Oregon_Kmeans_v1 /vol/v1/proj/LTOP_Oregon/rasters/02_Kmeans/gee/





#### **Merge GEE tiff chunks** 

	2. 1. Location

		/vol/v1/proj/LTOP_Oregon/rasters/02_Kmeans/gee

	1. Image 

		gdal_merge.py *.tif -o ../LTOP_Oregon_Kmeans_cluster_image.tif


#### **Sample Kmeans raster** (QGIS - Sample Raster Values)

	1. Qgis (TOOL: Sample Raster Values)

		a)Input 

			/vol/v1/proj/LTOP_Oregon/rasters/02_Kmeans/LTOP_Oregon_Kmeans_cluster_image.tif

			/vol/v1/proj/LTOP_Oregon/vectors/01_SNIC/01_snic_seed_pixel_points/01_snic_seed_pixel_points.shp

		b) Output column prefix

			cluster_id

		c) output

			/vol/v1/proj/LTOP_Oregon/vectors/02_Kmeans/LTOP_Oregon_Kmeans_Cluster_IDs.shp




#### **Get single point for each cluster**

	1) location

		/vol/v1/proj/LTOP_Oregon/scripts/kMeanClustering/randomDistinctSampleOfKmeansClusterIDs_v2.py

	2) Edit in script parameters  

		a) input shp file:

			/vol/v1/proj/LTOP_Oregon/vectors/02_Kmeans/LTOP_Oregon_Kmeans_Cluster_IDs/LTOP_Oregon_Kmeans_Cluster_IDs.shp

		b) output shp file:

			/vol/v1/proj/LTOP_Oregon/vectors/02_Kmeans/LTOP_Oregon_Kmeans_Cluster_ID_reps/LTOP_Oregon_Kmeans_Cluster_IDs.shp

	3) conda 

		conda activate geo_env

	4) run script


#### Upload SHP file of 5000 Kmeans cluster IDs points to GEE

	1) location 

		/vol/v1/proj/LTOP_Oregon/vectors/02_Kmeans/
	
	2) zip folder 

		zip -r LTOP_Oregon_Kmeans_Cluster_ID_reps.zip LTOP_Oregon_Kmeans_Cluster_ID_reps/

	3) upload to GEE 

		users/emaprlab/LTOP_Oregon_Kmeans_Cluster_ID_reps

#### **Abstract sample of Landsat Collections with 5000 Kmeans cluster reps** (GEE)

	1) Edit/Review parameters in 03abstractSampler.js

	2) Run 03abstractSampler.js in GEE

	3) output

		LTOP_Oregon_Abstract_Sample_annualSRcollection_Tranformed_NBRTCWTCGNDVIB5_v1.csv		

#### Download CSV from Google Drive

	1) Download

		LTOP_Oregon_Abstract_Sample_annualSRcollection_Tranformed_NBRTCWTCGNDVIB5_v1.csv

	2) location (islay)

		/vol/v1/proj/LTOP_Oregon/tables/abstract_sample_gee


#### Create Abstract image with CSV (python) 

	1. Input


	2. Outputs

		a) image directory



		b) SHP directory



	3. Run Command  





3. Upload rasters to GEE 

4. 1. ...

5. Make image collection from rasters 

6. 1. ...

7. Run Abstract imager for each index 

8. 1. ...

9. Download CSV ‘s one for each index 

10. 1. ...

11. Run scoring scripts

12. 1. ..

13. Upload LTOP selected csv to GEE

14. 1. …

15. Run GEE …

16. 1. ...

