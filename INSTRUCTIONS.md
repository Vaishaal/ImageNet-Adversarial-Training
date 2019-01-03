
## Dependencies:

+ TensorFlow≥1.6 with GPU support
+ OpenCV 3
+ Tensorpack==0.9.1
+ horovod≥0.15 with NCCL support
  + horovod has many [installation options](https://github.com/uber/horovod/blob/master/docs/gpus.md) to optimize its multi-machine/multi-GPU performance.
    You might want to follow them.
+ TensorFlow [zmq_ops](https://github.com/tensorpack/zmq_ops) (needed only for training)
+ ImageNet data in its [standard directory structure](https://tensorpack.readthedocs.io/modules/dataflow.dataset.html#tensorpack.dataflow.dataset.ILSVRC12)


## Model Zoo:

| Model                                      | error rate w/ <br/> clean images | error rate & attack success rate <br/> against 10-step PGD | error rate & attack success rate <br/> against 100-step PGD | error rate & attack success rate <br/> against 1000-step PGD | Flags                               |
|:-------------------------------------------|:--------------------------------:|:----------------------------------------------------------:|:-----------------------------------------------------------:|--------------------------------------------------------------|:-----------------------------------:|
| Res152 Baseline [:arrow_down:]()           | 8%                               | 3%                                                         | 3%                                                          |                                                              | `--arch ResNet -d 152`              |
| Res152 Denoise    [:arrow_down:]()         | 6%                               | 4%                                                         | 4%                                                          |                                                              | `--arch ResNetDenoise -d 152`       |
| ResNeXt101 Dense Denoise  [:arrow_down:]() | 5%                               | 7%                                                         | 7%                                                          |                                                              | `--arch ResNeXtDenseDenoise -d 101` |

Note:

1. As mentioned in the paper, our attack scenario is: 

   1. targeted attack with random uniform target label 
   2. maximum perturbation per pixel is 16.
   
   We do not perform untargeted attack, nor do we let the attacker choose the target label, 
   because we believe such tasks are not realistic on the 1000 ImageNet classes.

2. For each (attacker, model) pair, we provide both the error rate of our model, 
   and the attack success rate of the attacker, on the ImageNet validation set. 
   A target attack is considered successful if the image is classified to the target label.

   If you develop a new robust model, please compare its error rate with our models.
   Don't compare the attack success rate, because then the model can cheat by making random predictions.

   If you develop a new attack method against our models,
   please compare its attack success rate with PGD.
   Don't compare the error rate, because then the method can cheat by becoming
   untargeted attacks.

3. The "ResNeXt101 Dense Denoise" is the submission that won the champion of
   black-box defense track in [Competition on Adversarial Attacks and Defenses 2018](https://en.caad.geekpwn.org/).


## Evaluate White-Box Robustness:

To evaluate on one GPU, run this command:
```
python main.py --eval --load /path/to/model_checkpoint --data /path/to/imagenet \
  --attack-iter [INTEGER] --attack-epsilon 16.0 [--architecture-flags] 
```

The "architecture flags" for the pre-trained models are available in the model zoo.
`attack-iter` can be set to 0 to evaluate its clean image error rate.

Using a K-step attacker can make the evaluation K-times slower.
To speed up evaluation, run it under MPI with multi-GPU or multiple machines, e.g.:

```
mpirun -np 8 python main.py --eval --load /path/to/model_checkpoint --data /path/to/imagenet \
  --attack-iter [INTEGER] --attack-epsilon 16.0 [--architecture-flags] 
```


## Evaluate Black-Box Robustness:

We provide a command line option to produce predictions for an image directory:
```
python main.py --eval-directory /path/to/image/directory --load /path/to/model_checkpoint \
  [--architecture-flags]
```

This will produce a file "predictions.txt" which contains the filename and
predicted label for each image found in the directory.
You can use this to evaluate its black-box robustness.

## Train:

Adversarial training takes a long time and we recommend doing it only when you have a lot of GPUs.
You can use our code for standard ImageNet training as well (with `--attack-iter 0`).

To train, first start the data serving process __once on each machine___:
```
$ ./serve-data.py --data /path/to/imagenet/ --batch 32
```

Then, launch a distributed job with MPI. You may need to consult your cluster
administrator for the MPI command line arguments you should use. 
On a cluster with InfiniBand, it may look like this:

```
 mpirun -np 16 -H host1:8,host2:8 --output-filename train.log \
    -bind-to none -map-by slot -mca pml ob1 \
    -x NCCL_IB_CUDA_SUPPORT=1 -x NCCL_IB_DISABLE=0 -x NCCL_DEBUG=INFO \
    -x PATH -x PYTHONPATH -x LD_LIBRARY_PATH \
    python main.py --data /path/to/imagenet \
        --batch 32 --attack-iter [INTEGER] --attack-epsilon 16.0 [--architecture-flags]
```

If your cluster is managed by SLURM, we provide some sample slurm job scripts
(see `train.slurm.sh` and `eval.slurm.sh`) for your reference. 

The training code will also perform distributed evaluation of white-box robustness.