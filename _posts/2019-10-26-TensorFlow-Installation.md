---
layout: post
title: TensorFlow Installation
categories: 
 - BigData
permalink: /posts/5
nocomments: true  # Disable the comment box. This is EasyBook feature
---

TensorFlow is a powerful open source software library for numerical computation, particular well suited and fine-tuned for large-scale Machine Learning. Its basic principle is simple: you first define in Python a graph of computations to perform, and then TensorFlow takes that graph and runs it efficiently using optimizing C++ code. In this blog, I would like to install TensorFlow whether using macOS or linux, in order to continue operating on TensorFlow. 

## Environment for TensorFlow ##

Operating system：macOS Catalina

TensorFlow version: 1.15.0

Python version: 3.7.4

## Procedure of the operation ##

1. Opening the terminal, Install **pip** and **virtualenv** with the following code:

   `sudo easy_install pip3`

   `pip3 install --upgrade virtualenv`

2. Creating the virtualenv environment with the following code

   `Virtualenv –system-site-package -p python3 tartgetDirectory #cast to python 3.n`

   The tartgetDirectory represents the top path of the virtualenv content tree. In this situation, I choose the path to be /Users/haoyingkai/tensorflow

3. Activating the virtualenv environment with the following code:

   `cd /Users/haoyingkai/tensorflow/`

   `source ./bin/activate #using bash, sh, ksh, zsh  `

   Now we are able to find that the notification of the command line has become to :

   `(tensorflow) MacBook-pro:tensorflow haoyingkai$`

4. Install **TensorFlow** and its all dependencies with the following code:

   `Pip3 install –-upgrade tensorflow`

5. Install **ipython**, **jupyter** with the following code:

   `Pip3 install –-upgrade ipython`

   `Pip3 install –-upgrade jupyter`

6. Start Jupiter notebook:

   `jupyter notebook`

7. Try some tensorflow programs in jupyter notebook like this, to check out whether installations are well finished:

```python
import tensorflow as tf

x = tf.Variable(3, name = "x")
y = tf.Variable(4, namr = "y")
f = x*x*y + y + 2

sess = tf.Session()
sess.run(x.initializer)
sess.run(y.initializer)
result = sess.run(f)
print(result)
sess.close()
```

The result is 42.

## Conclusion ##

1.  At the beginning of the installation, I was so confusing about the method on book. Since I did not understand what the “ML_PATH” means, I installed the tensorflow on the macOS without isolating it in a separate environment. Then I checked some methods on the internet. It is said on a blog that they highly recommend installing tensorflow by using virtualenv. Virtualenv is a virtual Python environment which is isolated from other Python environments. As a result, Python on the virtualenv will not be interrupted by other Python programs. 
2.  Using the virtualenv is not that easy. I installed the jupyter and ipython outside the virtualenv at first. Therefore, after I activating the tensorflow, I could not import the tensorflow on the jupyter notebook. Then, I tried to uninstall the jupyter and ipython outside and reinstalled them in virtualenv environment(Uninstalling the jupyter was very difficult. I had to uninstall the relevant jupyter dependencies one by one.). After that, I ran a program in the jupyter notebook which was worked sussessfully.
3. Now, TensorFlow is basically having two versions(1.n.n & 2.0.0). The codes on the textbook which we are familiar with are  printed by version 1.n.n. Thus, If we install the latest version as usual, code will show something different from book. For instance, "tf.Session( )" will be replaced by "tf.compat.v1.Session( )". So if you want to try something new, you can download version 2.0.0. However, if you want to learn from the textbook, I recommend you to download version 1.n.n.
4. Eventrally, I realize that docker is also an isolated environment. I will try to install TensorFlow on docker these days.

