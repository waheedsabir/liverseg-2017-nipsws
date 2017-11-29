# Detection-aided liver lesion segmentation

Here we present the implementation in TensorFlow of our [work]() about liver lesion segmentation accepted in the [Machine Learning 4 Health Workshop](https://ml4health.github.io/2017/) of NIPS 2017. Check our [project](https://imatge-upc.github.io/liverseg-2017-nipsws/) page for more information.

**In order to develop this code, we used [OSVOS](https://github.com/scaelles/OSVOS-TensorFlow) and modified it to suit it to the liver lesion segmentation task.** 

![Architecture of the network](https://github.com/imatge-upc/medical-2017-liver/blob/master/img/architecture.png?raw=true)

 In this work we propose a method to segment the liver and its lesions from Computed Tomography (CT) scans using Convolutional Neural Networks (CNNs), that have proven good results in a variety of computer vision tasks, including medical imaging. The network that segments the lesions consists of a cascaded architecture, which first focuses on the region of the liver in order to segment the lesions on it. Moreover, we train a detector to localize the lesions, and mask the results of the segmentation network with the positive detections. The segmentation architecture is based on DRIU~\cite{maninis2016deep}, a Fully Convolutional Network (FCN) with side outputs that work on feature maps of different resolutions, to finally  benefit from the multi-scale information learned by different stages of the network. The main contribution of this work is the use of a detector to localize the lesions, which we show to be beneficial to remove false positives triggered by the segmentation network. 
 

## Publication

Our workshop paper is available on [arXiv](), and related slides [here](https://www.slideshare.net/xavigiro/detectionaided-liver-lesion-segmentation-using-deep-learning).

If you find this code useful, please cite with the following Bibtex code:

````
to-be-published
````


## Code Instructions

### Installation

1. Clone this repository

````
git clone https://github.com/imatge-upc/liverseg-2017-nipsws.git
````

2. Install if necessary the required dependencies:

* Python 2.7
* [Tensorflow](https://www.tensorflow.org/install/) r1.0 or higher 
* Python dependencies: PIL, numpy, scipy

If you want to test our models, download the different [weights](https://mega.nz/#!1r4XmDaB!9SPxHxMh1eYGzxtJzKN-CTkODibYOxkxjoBY2gWVaFQ). Extract the contents of this folder in the root of the repository, so there is a ````train_files```` folder with the following checkpoints:

* Liver segmentation checkpoint
* Lesion segmentation checkpoint
* Lesion detection checkpoint

If you want to train the models by yourself, we provide also the following pretrained models:

* VGG-16 weights
* Resnet-50 weights weights


### Data

This code was developed to participate in the [Liver lesion segmentation challenge (LiTS)](https://competitions.codalab.org/competitions/17094), but can be used for other segmentation tasks also. The LiTS database consists on 130 CT scans for training and 70 CT scans for testing. In the training set, we did our own partition, being CT scans with ids lower than 105 being used for training and the remanining for testing. They are compressed in a nifti format. 

The code expects that the database is inside the ````LiTS_database```` folder. Inside there should be the following folders:

* ```images_volumes```: inside there should be a folder for each CT volume. Inside each of these folders, there should be
a .mat file for each CT slice of the volume. The preprocessing required consists in clipping the values outside the range (-150,250) and doing max-min normalization.
* ```liver_seg```: the same structure as the previous, but with .png for each CT slice with the mask of the liver.
* ```item_seg```: the same structure as the previous, but with .png for each CT slice with the mask of the lesion.

An example of the structure for a single slice of a CT volume is the following:

```
LiTS_database/images_volumes/31/100.mat
LiTS_database/liver_seg/31/100.png
LiTS_database/item_seg/31/100.png
```

We provide a file in matlab to convert the nifti files into this same structure. In our case we used this [matlab library](https://ch.mathworks.com/matlabcentral/fileexchange/8797-tools-for-nifti-and-analyze-image). You can use whatever library you decide as long as the file structure and the preprocessing is the same. 

```bash
cd /utils/matlab_utils
matlab process preprocess_database_liver.m
```

### Liver segmentation

**1. Train the liver model**

In seg_liver_train.py you should indicate a dataset list file. An example is inside ```seg_DatasetList```, ```training_volume_3.txt```. Each line has:

```img1 seg_lesion1 seg_liver1 img2 seg_lesion2 seg_liver2 img3 seg_lesion3 seg_liver3``` 

If you just have segmentations of the liver, then repeat ```seg_lesionX=seg_liverX```. If you used the folder structure explained in the previous point, you can use the training and ```testing_volume_3.txt``` files.

```bash
python seg_liver_train.py
```

**2. Test the liver model**

A dataset list with the same format but with the test images is required here. If you don't have annotations, simply put a dummy annotation X.png. There is also an example in ```seg_DatasetList/testing_volume_3.txt```. 

```bash
python seg_liver_test.py
```

### Lesion detection

This network samples locations around liver and detects whether they have a lesion or not. 

**1. Crop slices around the liver**

In order to train the algorithm, first we crop the images around the liver region. If you don't have the cropped slices, you can use the following script:

```bash
cd utils/crops_methods
python compute_3D_bbs_from_gt_liver.py
```

This will generate three folders:

```
bb_liver_seg_alldatabase3_gt_nozoom_common_bb
bb_liver_lesion_seg_alldatabase3_gt_nozoom_common_bb
bb_images_volumes_alldatabase3_gt_nozoom_common_bb
```

The default version will generate the cropped slices around the liver segmentation mask in the ground truth, but you can change it so that in considers liver predictions.

**2. Sample locations around liver**

The first step is to sample these locations. We need a .txt with the following format:

```img1 x1 x2 id```

Example:

```images_volumes/97/444 385.0 277.0 1```

```x1``` and ```x2``` are the coordinates of the upper-left vertex of the bounding box and ```id``` is the data augmentation option.  There are two options in this script. To sample locations for slices with ground truth or without. In the first case, two separate lists will be generated, one for positive locations (/w lesion) and another for negative locations (/wo lesion), in order to train the detector with balanced batches. These lists are already generated so you can use them, they are inside ```det_DatasetList``` (for instance, ```training_positive_det_patches_data_aug.txt``` for the positive patches of training set).

In case you want to generate other lists, use the following script:

```bash
cd utils/sampling
python sample_bbs.py
```

**3. Train lesion detector** 

Once you have sampled the positive and negative locations, or use the default lists, you can use the following command to train the detector.

```bash
python seg_lesion_train.py
```

**4. Test lesion detector** 

In order to test the detector, you can use the following command:

```bash
python seg_lesion_test.py
```

This will create in the folder ```detection_results``` a folder with the name of the id given to the experiment, and inside two .txt files, one with the hard results (considering a th of 0.5) and another with soft results with the prob predicted by the detector that a location is unhealthy.

### Lesion segmentation


**1. Train the lesion model**

In order to train the algorithm that does not backpropagate through pixels outside the liver, each line of the list.txt file should have the following format.

```img1 seg_lesion1 seg_liver1 result_liver1 img2 seg_lesion2 seg_liver2 result_liver1 img3 seg_lesion3 seg_liver3 result_liver1 ```

An example list file is ```seg_DatasetList/training_lesion_commonbb_nobackprop_3.txt```. If you used the folder structure proposed in the Database section, and you have named the folders of the cropped slices as proposed in the ```compute_3D_bbs_from_gt_liver.py``` file, you can use the ```seg_DatasetList/training_lesion_commonbb_nobackprop_3.txt``` and the one for testing. 

```bash
python seg_lesion_train.py
```

**2. Test the lesion model**

The command to test the network is the following:

```bash
python seg_lesion_test.py
```

In this case, observe that the script does 4 different steps
1. Do inference with the network
2. Return results to the original size (from cropped slices to 512x512)
3. Mask the results with the liver segmentation masks 
4. Check positive detections of lesions in the liver. Remove those false positive of the segmentation network using the detection results.  (You need the detection results, obtained in the next point!)


## Contact

If you have any general doubt about our work or code which may be of interest for other researchers, please use the [public issues section](https://github.com/imatge-upc/liverseg-2017-nipsws/issues) on this github repo. Alternatively, drop us an e-mail at <miriam.bellver@bsc.es>.
