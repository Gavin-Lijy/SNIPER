# README for SNIPER
This is the hg38 branch of the SNIPER implementation. We have updated this branch to use newer python libraries (please see requirements for more details). Please email kxiong@andrew.cmu.edu with any questions about installation or usage.

# Installation

To quickly install SNIPER, clone SNIPER's repository and install the necessary requirements by running

`pip install -r requirements.txt`

in the shell. We recommend creating a separate python 3.8 environment. Installation should take under 15 minutes for a computer with broadband internet connection.

## Requirements

All Python dependencies can be installed by running

`pip install -r requirements.txt` (see Installation)

### Python dependencies:

* Python (3.8). SNIPER's hg38 build has been confirmed to work on 3.8. Python 3.6 and 3.7 may work as well, but some packages may not support 3.6 and 3.7 in our listed versions.
* Tensorflow (2.4.0)
* h5py (2.10.0)
* Keras (2.4.3)
* numpy (1.19.4)
* scipy (1.5.4)

### Juicer Tools

Requires Java Runtime Engine (on Linux, default-jre should work).

[Juicer tools](https://github.com/aidenlab/juicer/wiki/Juicer-Tools-Quick-Start) is a utility developed by the Erez Lieberman-Aiden lab that can extract Hi-C data from [.hic files](https://github.com/aidenlab/juicer/wiki/Pre). Follow the link provided to download `juicer_tools.jar`. Make a note of where `juicer_tools.jar` is stored on your file system.

SNIPER will call the directory where `juicer_tools.jar` is stored.

### CUDA and cuDNN:

SNIPER was developed using a NVidia GeForce GTX 1080 Ti. SNIPER hg38 uses [CUDA 10.1](https://developer.nvidia.com/cuda-90-download-archive) and [cuDNN](https://developer.nvidia.com/cudnn) v7.6.5 as required by Tensorflow 2 to run Keras on the Tensorflow backend. Please email kxiong@andrew.cmu.edu with any questions regarding python and CUDA environments.

Because there are so many versions of NVidia GPUs, we cannot say for certain how long SNIPER will need to finish training. For reference, SNIPER takes approximately 15 seconds to train one epoch of its autoencoder on our 1080 Ti. For users without a dedicated NVidia GPU, SNIPER will take significantly longer to train. Using a 3.6 GHz 6-core/12-thread Intel 8th-generation processor, one epoch took approximately 3 minutes.

# Usage

SNIPER is separated into two modules - training and application. To **train** a new SNIPER model, run the following python command:

`python sniper_train.py <input_hic_path> <target_hic_path> <annotation_path> [options]`

`input_hic_path` is the file path to the .hic file of the downsampled training Hi-C matrix. `target_hic_path` is the path to the .hic file of the dense target Hi-C matrix. We have provided GM12878's ground truth annotations in .mat format in SNIPER's root directory. `annotation_path` is the path to a .mat file of the GM12878 annotations published by Rao et al. (2014). We have included a .mat file of their annotations in the `data` directory of this repository (`data/labels.mat`).

Training with `.hic` files from scratch can take up to 60 minutes (depending on the size of the `.hic` file) because Juicer needs to extract the inter-chromosomal data of each pair of odd and even chromosomes, convert the contact data into a matrix, and then trim the matrix. Once training is complete, `sniper_train.py` will output six models to the user's specified directory (see Command line options). These models are the autoencoders, encoders, and classifiers trained on odd and even chromosomes.

To **apply** SNIPER to another cell line, run the following python command:

`python sniper.py <input_path> <output_path> <odd_encoder> <odd_clf> <even_encoder> <even_clf> [options]`

`input_path` specifies the path to the input Hi-C matrix (`.hic` or `.mat` format). `output_path` specifies a `.mat` file of the output predictions. `[odd/even]_encoder` specifies the keras model of the encoders trained with odd or even chromosomes aligned along the rows. `[odd/even]_clf` specifies the keras classifier model trained with odd or even chromosomes along the rows.

Application of SNIPER can likewise take up to 60 minutes to run because of Juicer's extraction process. `sniper.py` will output a `.mat` file whose keys `odd_predictions` and `even_predictions` refer to odd and even chromosome predictions respectively. In addition, SNIPER will output a `.bed` file in 100kb resolution with chromosomal coordinates of each prediction. The bed file is formatted similarly to the subcompartment predictions bed file in Rao et al. 2014 with color coding for easier visualization on the genome browser. To view the bed file on the genome browser, an additional header must be added as the first line of the file in the following format:

`track name='<track_name>' description='<description>' itemRgb='On'`

Pre-computed SNIPER hg38 models can be found here:

https://cmu.app.box.com/s/0f2c22e3ptptd0hxme6eez05tlb8ba85

## Command line options:

`-c` Specify a custom crop map directory that contains a crop map and crop indices. A crop map maps chromatin loci after the original matrix was trimmed to original chromatin locations before removing sparse loci, loci labeled as NA, and loci labeled as B4. Crop map format:

* N-by-3 2D array where N corresponds to the number of rows in the odd or even training matrices.
* Column 1 - index of a locus along the rows or columns of the input matrix before trimming
* Column 2 - chromosome of a locus
* Column 3 - position within the chromosome in units of 100kb

The row and column crop maps we used are provided in the `crop_map` directory.
Crop indices specify which rows and columns are sparse or labeled as NA or B4 in Rao et al's annotation set, which are removed from the input matrix. The general rule of thumb is to remove rows and columns where more than 30% of entries are zeros.

`-dd` Specifiy a directory to store output files:
* .txt files from juicer_tools if they aren't flagged for automatic removal
* .mat files of Hi-C matrices if the `-sm` flag is set
* Autoencoder, encoder, and classifier keras models
By default, SNIPER will store all files to its installation directory. We recommend setting this flag to avoid clutter in the installation directory and prevent unnecessary drive strain if SNIPER is installed on a solid state drive.

`-jt` Specify the path to `juicer_tools.jar`

`-sm` Turn this flag on to store .mat files of Hi-C matrices. Doing so will occupy approximately 3.2 GB of disk space but save a substantial amount of time if the pipeline abruptly terminates for some reason and has to be re-run.

`-ar` Turn this flag on to automatically remove .txt files output by Juicer Tool. Doing so will prevent clutter on the hard drive. Leaving this flag off will save time on subsequent training runs. We recommend turning this flag on if running multiple training instances on different cell types in the same directory.

`-ow` Turn this flag on to overwrite data existing on in the `-dd` directory. Recommended if running a new training instance on different cell types in a directory with existing data.

# Data Availability

`.hic` files we used for training can be found at:

https://cmu.box.com/s/qp2we42r7bdghimqxxbj9vhbsn4eue0c

Of the included files, `GM12878_hg38.hic` is the high-coverage Hi-C data used for training and was converted from Rao 2014's hg19 data to hg38 using [LiftOver](http://hgdownload.soe.ucsc.edu/admin/exe/linux.x86_64/liftOver). `GM12878_rao_hg38_<ds>.hic` are the downsampled GM12878 Hi-C data where `<ds>` refers to the downsample level, i.e. 0.1 denotes 10% of the contacts present in `GM12878_hg38.hic`.

# Citation

If you use SNIPER in your work, please cite:

Xiong, K., Ma, J. Revealing Hi-C subcompartments by imputing inter-chromosomal chromatin interactions. Nat Commun 10, 5069 (2019).
