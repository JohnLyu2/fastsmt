[FastSMT](https://fastsmt.ethz.ch/) is a tool to augment the Z3 SMT solver by learning to optimize its performance for specific dataset of formulas. This repo is a fork of the [original repo](https://github.com/eth-sri/fastsmt), maintained by the authors of [Z3alpha](https://github.com/JohnLyu2/z3alpha), for the purpose of compatibility with the latest Z3 and other enviornments.

## Setup Instructions

Clone this repository and navigate under the repo root

Download Z3 4.12.2

```bash
$ git clone https://github.com/Z3Prover/z3.git z3
$ cd z3

# Checkout Z3 version 4.12.2 that we tested against
$ git checkout tags/z3-4.12.2
```

Install and compile Z3 4.12.2 (with cpp bindings):

```bash
$ python scripts/mk_make.py 
$ cd build
$ make # (optional) use `make -j4` where 4 is the number of threads used to compile Z3, will likely take couple of minutes
$ sudo make install
$ cd ../..
``` 

Setup python virtual environment. 

```bash
$ virtualenv -p python3 --system-site-packages venv
$ source venv/bin/activate
(venv) $ python setup.py install
```

Install and compile Z3 4.12.2 (with Python bindings):

```bash
(venv) $ cd z3
# To generate correct python bindings make sure you activated the virtual env before Z3 compilation
(venv) $ python scripts/mk_make.py --python
(venv) $ cd build
(venv) $ make # (optional) use `make -j4` where 4 is the number of threads used to compile Z3, will likely take couple of minutes
(venv) $ sudo make install
(venv) $ cd ../..
``` 

Finally, compile C++ runner: 
```bash
$ cd fastsmt/cpp
$ make -f make_z3
$ cd ..
```

Please run the tests to make sure installation was successful:

```bash
$ ./test/run_tests.sh
OK!
```

## Creating the dataset

In this section we describe how to prepare the data for synthesis procedure. Our running example will be *leipzig* formulas (QF_NIA theory) from SMT-LIB benchmarks. First, download the formulas into `examples` subdirectory.

```bash
$ mkdir examples
$ cd examples
$ wget http://smt-lib.loria.fr/zip/QF_NIA.zip
$ unzip QF_NIA.zip
$ mkdir QF_NIA/leipzig/all
$ mv QF_NIA/leipzig/*.smt2 QF_NIA/leipzig/all/
$ cd ..
```

You should always create `all` subdirectory in which you should put all of the formulas in SMT2-LIB format.
If you want to split your dataset into training/validation/test set in the ratio 50/20/30 then you use the following command:

```bash
(venv) $ python scripts/py/dataset_create.py --split "50 20 30" --benchmark_dir examples/QF_NIA/leipzig
```

In subfolders `examples/QF_NIA/leipzig/train`, `examples/QF_NIA/leipzig/valid`, `examples/QF_NIA/leipzig/test` you can find formulas which belong to training, validation and test set. 

## Learning and synthesis

Synthesis script performs search over the space of possible strategies and tries to synthesize best strategy for each instance.
It receives configuration for the synthesis procedure, benchmark directory and other information. 

Help menu with information about all possible arguments can be accessed with:

```bash 
(venv) $ python synthesis/learning.py -h
```

In this toy example, we are going to load formulas from leipzig benchmark and run our procedure to synthesize best strategy for every formula. Concretely, we will first search randomly for 5 iterations, then train the neural network model and search for 5 more iterations (this time guided by the model). 
Here is an example of the full command (execution should take few minutes):

```bash
(venv) $ python synthesis/learning.py experiments/configs/leipzig/config_apprentice.json \
                --benchmark_dir examples/QF_NIA/leipzig/ \
                --max_timeout 10 \
                --num_iters 5 \
                --iters_inc 5 \
                --pop_size 1 \
                --eval_dir eval/synthesis/ \
                --smt_batch_size 100 \
                --full_pass 2 \
                --num_threads 10 \
                --experiment_name leipzig_example
```

Learned strategies are saved in directory given as `eval_dir`. As a guideline for setting the parameters of the learning procedure, we suggest to look at our experiments in the `experiments` subfolder. 

Now, best strategies are saved in `eval/synthesis/leipzig_example/train/2/strategies.txt`.
For the sake of this experiment, to see the results faster, we will sample only 10 strategies from the list of best strategies:

```bash
cat eval/synthesis/leipzig_example/train/2/strategies.txt | shuf | head -10 > sample_strategies.txt
```

You can also use `eval/synthesis/leipzig_example/train/2/strategies.txt` instead of `sample_strategies.txt` in the following command, but be prepared to wait more. Next, we are going to synthesize a final strategy in SMT2 format from these 10 learned strategies (running the script should take some time):

```bash
(venv) $ python synthesis/multi/multi_synthesis.py \
         --cache cache_leipzig_multi.txt \
         --max_timeout 10 \
         --benchmark_dir examples/QF_NIA/leipzig/ \
         --num_threads 10 \
         --strategy_file output_strategy.txt \
         --leaf_size 4 \
         --num_strategies 5 \
         --input_file sample_strategies.txt
```

After this command is finished you can find the synthesized strategy in SMT2 format in file `output_strategy.txt`. This strategy can be passed to Z3 solver as explained in: https://rise4fun.com/z3/tutorial/strategies

## Validation

In order to evaluate final strategy synthesized by our system we provide a validation script. For an input, this script receives dataset with SMT2 formulas and a strategy. It runs Z3 solver with and without using given strategy and outputs the performance comparison (in terms of number of solved formulas and runtime).

```bash
(venv) $ python scripts/py/validate.py \
        --strategy_file output_strategy.txt \
        --benchmark_dir examples/QF_NIA/leipzig/test \
        --max_timeout 10 \
        --batch_size 4
```

## Additional information

In order to reproduce experimental results from our paper, consult README file in `fastsmt/experiments` subfolder. For more details on the configuration of our system, consult README in `fastsmt/experiments/configs`.












