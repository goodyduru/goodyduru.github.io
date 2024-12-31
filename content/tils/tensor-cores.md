+++
date = '2024-12-31T15:01:35+01:00'
title = 'Tensor Cores'
+++
I had always thought that the difference between high end CPUs vs GPUs were between 3x to 20x. It turns out that I was totally wrong. Modern GPUs can have up to 100x better performance than CPUs on Machine Learning workloads. This is because of **Tensor Cores**.

[Tensor Cores](https://www.digitalocean.com/community/tutorials/understanding-tensor-cores) are parts of the GPUs that specialize in Fused-Multiply Add operations `(a*b + c)`. These operations are common in Machine learning workloads.

I had always wondered why the idea of using lots of CPUs to replace GPUs in ML training and inference isn't widely used considering the price differences. It turns out Tensor Cores are a big part of that.