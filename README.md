# Segmenting Buildings for Disaster Resilience
## DSC 672 - Masters in Data Science Capstone Project
Authors:
* John Hoff
* Brian Pirlot
* Sebastian Zdarowski

This project was undertaken as a capstone for the Masters in Data Science program at DePaul University in Chicago.  The project was conducted over a period of 10 weeks.  Please see the included `DSC672_Group4_ProjectPresentation.pdf`, `DSC672_Group4_ProjectPaper.pdf`, and `DSC672_Group4_ProjectSupplement.pdf` for additional information.

This repository is intended to demonstrate the machine learning code that was developed for this capstone.  This was the live repository that was used for collaboration and cloud deployment.

## Initial Configuration

This project makes use of two independant Python virtual environments. One for [Solaris](https://github.com/CosmiQ/solaris) and one for [Mask R-CNN](https://github.com/matterport/Mask_RCNN).  These environments have not been unified due to the steps necessary to install Solaris.  The package and some of its dependencies are new enough that they are not yet fully stable in pip and conda.

### Windows Installation

#### Solaris

```
> python -m virtualenv venv
> venv\scripts\activate
> pip install https://download.lfd.uci.edu/pythonlibs/q4hpdf1k/Shapely-1.6.4.post2-cp37-cp37m-win_amd64.whl
> pip install https://download.lfd.uci.edu/pythonlibs/q4hpdf1k/GDAL-3.0.3-cp37-cp37m-win_amd64.whl
> pip install https://download.lfd.uci.edu/pythonlibs/q4hpdf1k/Fiona-1.8.13-cp37-cp37m-win_amd64.whl
> pip install https://download.lfd.uci.edu/pythonlibs/q4hpdf1k/Rtree-0.9.3-cp37-cp37m-win_amd64.whl
> pip install https://download.lfd.uci.edu/pythonlibs/q4hpdf1k/rasterio-1.1.2-cp37-cp37m-win_amd64.whl
> pip install torch===1.3.1 torchvision===0.4.2 -f https://download.pytorch.org/whl/torch_stable.html
> pip install -r requirements.txt
```

#### Mask R-CNN

Be sure to use the `mcrnn-requirements-gpu.txt` file to make use of a local GPU.

```
> python -m virtualenv mrcnn_venv
> venv\scripts\activate
> pip install -r mrcnn-requirements.txt
```

### OSX Installation

#### Solaris

OSX installation does make use of both conda and pip.  Both GDAL and pytorch are unfortunately not fully working with solaris on the versions that will be installed by default.  For instance, you need gdal==3.0.2 to install solaris, but need gdal==2.3.3 in order to get everything to _run_.

```
> conda create --prefix ./venv python=3.7
> conda activate ./venv
> conda install gdal
> conda install pytorch torchvision -c pytorch
> pip install solaris
> conda install gdal==2.3.3
> conda install pytorch torchvision -c pytorch-nightly
> pip install pystac
> pip install rio-tiler
> pip install scikit-learn
```

#### Mask R-CNN

```
> python -m virtualenv mrcnn_venv
> pip install -r mrcnn-requirements.txt
> venv\scripts\activate
```

### Linux Instalation

Please refer to the [SageMaker Guide](SageMaker.md) for details on installing the required Python modules for linux.

## Sample Data Preparation

Some sample data is included in this repository so that configuration can be quickly checked _without_ having to download the full training and testing data files.  Simply run the following command from the Python virtual environment:

```
> python -m prepare.extract_training_images -ts sample_source_data/sample -ds temp_data/sample -s 256 -z 19
```

## Downloading Real Project Data
The data used for this project is too large to be included directly in this repository.  It is hosted in Amazon S3 by Driven Data.  The following files should be downloaded and extracted into the `raw_source_data` directory.

* [https://drivendata-public-assets.s3.amazonaws.com/train_tier_1.tgz](https://drivendata-public-assets.s3.amazonaws.com/train_tier_1.tgz) (32 GB)
* [https://drivendata-public-assets.s3.amazonaws.com/train_tier_2.tgz](https://drivendata-public-assets.s3.amazonaws.com/train_tier_2.tgz) (40 GB)
* [https://drivendata-public-assets.s3.amazonaws.com/test.tgz](https://drivendata-public-assets.s3.amazonaws.com/test.tgz) (9 GB)

Once that is done, the following directories should now be present in the `raw_source_data` directory:

* `train_tier_1`
* `train_tier_2`
* `test`

## Real Data Preparation

Once everything has been downloaded and extracted to the indicated folders, the training data can be created using the following commands:

```
> python -m prepare.extract_training_images -ts raw_source_data/train_tier_1 -ds temp_data/tier1 -s 256 -z 19
> python -m prepare.extract_training_images -ts raw_source_data/train_tier_1 -ds temp_data/tier1_lg -s 256 -z 20
> python -m prepare.extract_training_images -ts raw_source_data/train_tier_2 -ds temp_data/tier2 -s 256 -z 19
> python -m prepare.extract_training_images -ts raw_source_data/train_tier_2 -ds temp_data/tier2_lg -s 256 -z 20
```

This will create 256x256 pixel tiles at a zoom level of 19 and 20.

The testing data can be prepared using the following command:

```
> python -m prepare.resize_test_images -ts raw_source_data/test -ds temp_data/test -s 256
```

_Please note: Preparing the data will take a significant amount of time._

## Sample Training Dataset Creation

```
> python -m prepare.create_training_testing_split -c sample -p sample -n 160 -s 0.25 -a y
```

## Real Training Dataset Creation

The following command will create a pair of datasets.  The first dataset is intended to compare Solaris to Mask RCNN and includes data from both tier 1 and tier 2.  This dataset has an 80/20 training and testing split.

The second dataset is intended for content submission and includes data from tier 1 only.  This dataset has a 95/5 training and testing split.  The testing split is intended only to tune the thresholding for final contest submission.

```
> python -m prepare.create_training_testing_split -c tier1,tier1_lg,tier2 -p comparison -n 40000 -s 0.2 -a y
> python -m prepare.create_training_testing_split -c tier1,tier1_lg -p contest -n 40000 -s 0.05 -a y
```

_Please note: When creating these datasets, the mean and standard deviation of the pixels are being calculated.  This is an expensive process that may take a significant amount of time._

## Mask RCNN Model Training

Please note that after a model is trained, it should be moved out of the `temp_data/logs` directory and directly into the `temp_data` directory.  The Mask RCNN model will save each epoch of training into the logs folder.  This allows for easy restarting of training if--and unfortunately when--training crashes or is interrupted.

The Mask RCNN will train in three total stages beginning with only the head layers and extending the training depth each stage.

### Sample Data Training

```
python -m model.train_mrcnn_model -df temp_data/sample_mask_rcnn_training_testing.csv -s 80 -vs 10 -eh 40 -er 20 -ea 20
```

When training is finished, the model should be moved to `temp_data/sample_mask_rcnn_building_0080.h5`.

### Comparison Model Training

```
python -m model.train_mrcnn_model -df temp_data/comparison_mask_rcnn_training_testing.csv -s 2000 -vs 200 -eh 40 -er 20 -ea 20
```

When training is finished, the model should be moved to `temp_data/comparison_mask_rcnn_building_0080.h5`.

### Contest Submission Training

```
python -m model.train_mrcnn_model -df temp_data/contest_mask_rcnn_training_testing.csv -s 2000 -vs 200 -eh 40 -er 20 -ea 20
```

When training is finished, the model should be moved to `temp_data/contest_mask_rcnn_building_0080.h5`.

## Visualizing Mask RCNN Inferences

Inferencing on individual map tiles can be quickly done with the testing code.  This intented primarily for demonstration purposes and will display the map tile and resulting bounding box and mask overlay.

### Sample Model Inferences

```
python -m model.test_mrcnn_model -df temp_data/sample_mask_rcnn_training_testing.csv -mp temp_data/sample_mask_rcnn_building_0080.h5 -t demo
```

### Comparison Model Inferences

```
python -m model.test_mrcnn_model -df temp_data/contest_mask_rcnn_training_testing.csv -mp temp_data/contest_mask_rcnn_building_0080.h5 -t demo
```

### Contest Model Inferences

```
python -m model.test_mrcnn_model -df temp_data/contest_mask_rcnn_training_testing.csv -mp temp_data/contest_mask_rcnn_building_0080.h5 -t demo
```

## Scoring the Mask RCNN Model

Scoring the model can be done by calling the inferencing without generating the image previews and calculating the jaccard index for the set of images.

### Sample Model Scoring

```
python -m model.test_mrcnn_model -df temp_data/sample_mask_rcnn_training_testing.csv -mp temp_data/sample_mask_rcnn_building_0080.h5 -t score -th 0.9
```

### Comparison Model Scoring

```
python -m model.test_mrcnn_model -df temp_data/comparison_mask_rcnn_training_testing.csv -mp temp_data/comparison_mask_rcnn_building_0080.h5 -t score -th 0.9
```

### Contest Model Scoring

```
python -m model.test_mrcnn_model -df temp_data/contest_mask_rcnn_training_testing.csv -mp temp_data/contest_rcnn_building_0080_v4.h5 -t score -th 0.9
```

## Mask RCNN Contest Submission

```
python -m model.test_mrcnn_model -df temp_data/contest_mask_rcnn_training_testing.csv -mp temp_data/contest_mask_rcnn_building_0080.h5 -t predict -th 0.9
```

## License and Attribution

The following project includes source code and examples from the following projects:

### [Solaris](https://github.com/CosmiQ/solaris)

### [Mask R-CNN](https://github.com/matterport/Mask_RCNN)
