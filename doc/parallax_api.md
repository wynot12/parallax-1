# Parallax API
This document explains Parallax API in detail. If you are a beginner of Parallax, follow [quick_start](quick_start.md).

## parallel_run

`parallel_run` invokes the `run` function to run the `single gpu graph` in the distributed environment specified in the `resource_info` file with a specific communication method. This is either MPI or the PS style communication.
``` shell
def parallel_run(single_gpu_graph, run, resource_info, num_iterations, 
                 sync=True, parallax_config=None)
```
* single_gpu_graph: A complete TensorFlow graph that can run on a single device.
* run:  A function which runs the transformed graph in a distributed environment.
* resource_info: A file which contains resource information.
* num_iterations: The number of iterations to be run on each worker.
* sync: The training method(synchronous/asynchronous). `True` is the default.
* [parallax_config](#parallaxconfig): The minor configuration for executing Parallax.

### ParallaxConfig
`parallax_config`, which is used in `parallel_run`, is an instance of **ParallaxConfig**, the minor configuration(**SessionConfig**, **CheckPointConfig**, **CommunicationConfig**) for executing Parallax.

```shell
ParallaxConfig(run_option=None, average_sparse=False, sess_config=None, redirect_path=None, 
               communication_config=CommunicationConfig(), ckpt_config=CheckPointConfig())
```

* run_option:  A string(PS or MPI). If the option is None, the faster communication is selected automatically.
* average_sparse: A boolean. If True, sparse parameters are updated by the averaged gradients over all replicas. Otherwise, the sum of all gradients are used.
* sess_config: The session configuration used in `tf.train.MonitoredTrainingSession`
* redirect_path: A string. Optional path to redirect logs as files.
* communication_config: The communication configuration for PS and MPI.
	* PSConfig
		* protocol: Specifies the protocol to be used by the server. Acceptable values include `"grpc"`, `"grpc+gdr"`, `"grpc+verbs"`, etc. `"grpc"` is the default.
		* replicate_variables: Each GPU has a copy of the variables, and updates its copy after the parameter servers are all updated with the gradients from all servers. Only works with `sync=True`.
	* MPIConfig
		* use_allgatherv: Specifies whether to utilize OpenMPI `allgatherv` instead of NCCL `allgather`. `use_allgatherv=False` is recommended by Parallax.
		* mpirun_options: A string or list. Specifies the mpirun options, such as `-mca btl ^openib`. Empty string is the default.
* ckpt_config: The configuration for checkpoint.
	* ckpt_dir: The checkpoint directory to store/restore global variables.
	* save_ckpt_steps: The frequency, in number of global steps, that a checkpoint is saved using a default checkpoint saver.
	* save_ckpt_secs: The frequency, in seconds, that a checkpoint is saved using a default checkpoint saver.

Below code is an example of how to use **ParallaxConfig**.
```
ckpt_config = parallax.CheckPointConfig(ckpt_dir=FLAGS.ckpt_dir,
                                        save_ckpt_steps=save_ckpt_steps)
ps_config = parallax.PSConfig(replicate_variables=FLAGS.replicate_variables,
                              protocol=FLAGS.protocol)
mpi_config = parallax.MPIConfig(use_allgatherv=FLAGS.use_allgatherv,
                                mpirun_options=FLAGS.mpirun_options)
parallax_config = parallax.Config()
parallax_config.run_option = FLAGS.run_option,
parallax_config.average_sparse = False
parallax_config.redirect_path = FLAGS.redirect_path
parallax_config.communication_config = parallax.CommunicationConfig(ps_config, mpi_config)
parallax_config.ckpt_config=ckpt_config
...
parallel_run(..., parallax_config=parallax_config)
```
