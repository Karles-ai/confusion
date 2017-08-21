# Interfaces for Confusion Classifiers

## PyTorch Confusion

Simply include `confusion_pytorch` using `import confusion_pytorch`, and add the requisite loss. Check source at `confusion_pytorch/__init__.py` for details.

## Caffe Confusion - Prerequisites
The following dependencies must be satisfied for proper functionality:
- [Caffe](https://github.com/BVLC/caffe) at `CAFFE_ROOT`. Note: for feature extraction, you must use the version of Caffe available [here](https://github.com/abhimanyudubey/caffe) (this version is exactly updated with the BVLC Caffe, with a couple of edits to HDF5 Output Layers, and is straightforward to compile).
- Python dependencies include `matplotlib`, `h5py`, `lmdb`, `tqdm`. Install these by running:

`for req in $(cat requirements.txt); do pip install $req; done`

- Environment variable `CAFFE_ROOT` must be set to the path where Caffe resides. This can be done by - 

`export CAFFE_ROOT=/path/to/your/caffe/root`

Please note that these interfaces support only LMDB and ImageData layers at the time.
To view help on a specific task, choose the related interface:
- [Training](#training-networks-in-caffe)
- [Feature Extraction](#feature-extraction-in-caffe)
- [t-SNE Visualization](#t-sne-visualization)

## Training Networks in Caffe
The script `train.py` provides a very easy and straightforward interface to train neural networks using Caffe. 

### Architecture Requirements in Prototxt Files
Note that, before you train the network, the following layers constraints are satisfied in model and solver prototxts:
- Data layer called `data`.
- Label layer called `label`.
- Training performance metric (any monotonically increasing metric with performance would be fine, most commonly it's top-1 accuracy) called `accuracy_train` in the `TRAIN` phase, and duplicate layer `accuracy_test` for the `TEST` phase.
- Training optimization metric (any monotonically decreasing metric with performance would be fine, most commonly it's Cross-Entropy) called `loss_train` in the `TRAIN` phase, and duplicate layer `loss_test` for the `TEST` phase.
- Similar metric as Accuracy/Top-5 called `top5_train` for `TRAIN` phase, and `top5_test` for `TEST` phase.
- batch_size in the `TRAIN` phase should be even (for confusion loss).

Please take a look at the models present in `caffe/models/` for some example prototxts.

### Basic Usage
For a fixed solver/model combination, the simplest command is:

```
./train.py --model /path/to/model \
--solver /path/to/solver \
--log_directory /path/to/log/directory \
--snapshot /path/to/snapshot \
--gpu 0
```

This command will run Caffe training while storing snapshots in the specified directory, with the specified model (overriding the settings in the prototxt file), on GPU 0. It is preferred that absolute paths be used to avoid bugs.

### Logging
In the logging directory, you will find the following files:
```
log_directory /
		model.prototxt
		solver.prototxt
		train_perf.csv
		test_perf.csv
		plot.pdf
		log.txt
```

The CSV files report the variation of Top-1, Top-5 and Cross-Entropy for both training and test sets. The `log.txt` is the complete output of `[INFO, ERROR]` from Caffe. The `plot.pdf` is a collection of two plots - variation of train/test performance v/s iterations, and loss function v/s iterations. These plots are updated as the training progresses.

The `model.prototxt` and `solver.prototxt` files are generated by the code and are the final prototxts that are used during runtime.

### Other Usage

The command `./train.py --help` will provide additional help. Some examples are:

#### To Override Supplied Parameters in the Solver

You can replace parameters supplied in the solver during runtime. To do that, pass the new parameters as an string dictionary to the `--params` argument. For example, if we wished to change the `base_lr` to `0.001` and `momentum` to `0.85`, we can specify the command as:

```
./train.py --model /path/to/model \
--solver /path/to/solver \
--log_directory /path/to/log/directory \
--snapshot /path/to/snapshot \
--gpu 0 \
--params '{"base_lr" : 0.001, "momentum": 0.85}'
```

#### To Specify the Data Location

As above, the location for training data, validation data and the mean file can also be specified during runtime. Example:

```
./train.py --model /path/to/model \
--solver /path/to/solver \
--log_directory /path/to/log/directory \
--snapshot /path/to/snapshot \
--gpu 0 \
--data '{"train" : "/path/to/training/data", "test": "/path/to/testing/data", "mean_file": "/path/to/mean_file"}'
```

Any combination of these can be used, however all supplied paths must exist and be absolute.

#### To Specify Confusion Loss

Confusion Loss can be added on execution in a manner similar to supplying parameters, by feeding in a string dictionary. For example, to apply confusion loss to layer "l1" with weight `0.001` and "l2" with weight `0.05`, the command:

```
./train.py --model /path/to/model \
--solver /path/to/solver \
--log_directory /path/to/log/directory \
--snapshot /path/to/snapshot \
--gpu 0 \
--confusion '{"l1" : 1, "l2": 0.005}'
```

Note that by default the type of confusion that will be applied is pairwise. To use entropic confusion instead, add the argument `--entropic` to the command. Please make sure to adjust the loss weights accordingly. The optimal range of operation for pairwise loss is 0.1N to 0.2N (where N is the number of classes), and for entropic is 0.1-0.5 (tuning may be required, but is fairly robust to hyperparameter value). A sample entropic confusion command is:

```
./train.py --model /path/to/model \
--solver /path/to/solver \
--log_directory /path/to/log/directory \
--snapshot /path/to/snapshot \
--gpu 0 \
--confusion '{"l1" : 0.001, "l2": 0.05}'
--entropic
```

## Feature Extraction in Caffe

Caffe's Python interface is pretty slow at computing features, and the interface `feature_extractor.py` uses the fast 'test' phase of the Caffe executable along with HDF5 layers to extract features quickly (multiple layers are produced in the same forward pass), producing CSV files for each layer. Note, however, that this supports only ImageData and LMDB Data layers, with data layer named `data` and label layer named `label`.

### Basic Usage

The basic input requires the model definition, weights, required layers (as a string list), input file (or directory in case of LMDB) and output prefix. Example:

```
./feature_extractor.py --model /path/to/model/prototxt \
--weights /path/to/caffemodel \
--output /path/to/output/prefix \
--layers '["layer1", "layer2"]' \
--input /input/imagelist/or/lmdb/file 
```

Here, the input has to be of the same type as the model definition - a path to LMDB database if `Data` layer with backend LMDB is used, and ImageData list if `ImageData` layer is used. If you have a folder of images you want to extract features from, you can run the following command inside the target folder to generate the list:

`ls | xargs -I @ sh -c 'echo $(pwd)/@ 0' > /path/to/imagedata/list`

This command will generate the required imagedata list (with dummy 0 labels) at `/path/to/imagedata/list`. Make sure the data layer is `ImageData` for such an approach.

### Advanced Options

As with the Training interface, you can select GPUs, custom batch_sizes (to override model definition batch_size) and the option to write headers in the CSV file (off by default).

## t-SNE Visualization

The script at `tsne.py` provides an easy interface to generate t-SNE plots.

