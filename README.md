# KyotoNMT
## An implementation of Neural Machine Translation in Chainer.
This code mainly implements the Neural Machine Translation system described in (Bahdanau et al., 2015). Also known as "RNNSearch", or "Sequence-to-Sequence Modeling with Attention Mechanism", this is, as of 2016, the most commonly used approach in the recent field of Neural Machine Translation.

This implementation uses the Chainer Deep Learning Library (http://chainer.org/).

#Requirements:
* Python 2.7.x
* A recent version of Chainer (> 1.9). Install with:
 
        pip install chainer

* Optionally, the plotting libraries plotly and bokeh are used in some visualisation scripts:

        pip install plotly
        pip install bokeh

#Usage:

There are essentially three steps in the use of KyotoNMT: data preparation (`make_data.py`), training (`train.py`), evaluation (`eval.py`). Each of these scripts has many options that can be displayed by running them with the `--help` option.

## Data Preparation
The required training data is a sentence-aligned parallel corpus that is expected to be in two utf-8 text files: one for source language sentences and the other target language sentences. One sentence per line, words separated by whitespaces. Additionally, some validation data should be provided in a similar form (a source and a target file). This validation data will be used for early-stopping, as well as to visualize the progress of the training. One should also specify the maximum size of vocabulary for source and target sentences. For example:

    python make_data.py train.src train.tgt data_prefix --dev_src valid.src --dev_tgt valid.tgt  --src_voc_size 100000 --tgt_voc_size 30000

As a result of this call, two dictionaries indexing the 100000 and 30000 most common source and target words are created (with a special index for out-of-vocabulary words). The training and validation data are then converted to integer sequences according to these dictionaries and saved in a gzipped JSON file prefixed with `data_prefix`.

## Training
Training is done by invoking the `train.py` script, passing as argument the data prefix used in the data preparation part.

    python train.py data_prefix train_prefix

This simple call will train a network with size and features similar to those used in the original (Bahdanau et al., 2015) paper (except that LSTMs are used in place of GRUs). But there are many options to specify different aspects of the network: embedding layer size, hidden states size, number of lstm stacks, etc. The training settings can also be specified at this point: weight decay, learning rate, training algorithm, dropout values, minibatch size, etc. 

train.py will create several files prefixed by train_prefix. 
A JSON file train_prefix.config is created, containing all the parameters given to `train.py` (used for restarting an interrupted training session, or using a model for evaluation). 
A file train_prefix.result.sqlite is also created, containing a sqlite database that will keep track of the training progress. 

Furthermore, model files, containing optimized network parameters will be saved regularly. Every n minibatches (by default n = 200), an evaluation is performed on the validation set. Both perplexity and BLEU scores are computed. The BLEU score is computed by translating the validation set with a greedy search.

The models that have given the best BLEU and best perplexity so far are saved in files `train_prefix.model.best.npz` and `train_prefix.model.best_loss.npz` respectively. This allows to have early stopping based on two different criterions: validation BLEU and validation perplexity.

The sqlite database keep track of many information items during the training: validation BLEU and perplexity, training perplexity, time to process each minibatch, etc. An additional script `graph_training.py` can use this database to generate a plotly (https://github.com/plotly) graph showing the evolution of BLEU, perplexity and training loss. This graph can be generated while training is still in progress and is very useful for monitoring ongoing experiments. 

## Evaluation

Evaluation is done by running the script `eval.py`. It allows, among other things, to translate sentences by doing a beam search with a trained model. The following command will translate `input.txt` into `translations.txt` using the parameters that gave the best validation BLEU during training:

    ./eval.py train_prefix.config train_prefix.model.best.npz input.txt translations.txt--mode beam_search --beam_width 30

We usually find that it is better to use the parameters that gave the best validation BLEU rather than the ones that gave the best validation loss. Although it can be even better to do an ensemble translation with the two. The `eval.py` script has many options for tuning the beam search, ensembling several trained models, displaying the attention for each translation, etc.

TODO: describe UNK replacement

## Visualisation
1. Generate Training Graph

        #! shell
        pip install --user plotly
        python graph_training.py --lib plotly /path_to_experiment_dir/prefix_train/*.result.sqlite ~/public_html/graph.html

## Recommended Options

        #! shell
        python train.py /path_to_experiment_dir/prefix /path_to_experiment_dir/prefix_train --gpu 0
        --optimizer adam
        --weight_decay 0.000001
        --l2_gradient_clipping 1
        --mb_size 64
        --max_src_tgt_length 90
        // use smaller value if frequently running out of memory

