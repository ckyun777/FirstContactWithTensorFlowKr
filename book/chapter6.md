번역자: 김현서, Hyunseo Kim(boon0720@gmail.com)

6. PARALLELISM
6. 병렬화

The first TensorFlow package, appearing in November 2015, was ready to run on servers with available GPUs and executing the training operation simultaneously in them. In February 2016, an update added the capability to distribute and parallelize the processing.
2015년 11월에 출시된 첫 번째 TensorFlow 패키지는 훈련 조작(training operation)이 GPU와 서버에서 동시에 실행되었다. 추후 2016년 2월에 업데이트 되면서 배포(distribution)와 병렬처리 기능이 추가 되었다.

In this short chapter I’ll introduce how to use the GPUs. For those readers wanting to understand a little bit more of how these devices work, some references will be given in the last section, but. Given the introductory focus of this book, I’ll not enter in detail for the distributed versión, but for those readers interested some references will be given in the last section.
이번 장은 GPU 사용 방법을 소개한다. 마지막 부분에 제시된 일부 참조는 GPU가 어떻게 작동되는지 좀 더 알고자 하는 독자를 위한 것이다. 본 저서는 간략한 소개에 초점이 맞춰져 있으므로 배포 버전에 대한 세부적인 내용은 마지막 섹션에서 제공된 참조를 통해 도움 받을 수 있다.

Execution environment with GPUs
GPU 환경 실행

The TensorFlow package supporting GPUs requires the CudaToolkit 7.0 and CUDNN 6.5 V2. For installing the environment, we suggest to visit the cuda installation[44] website, for not going deep in details, also the information is up-to-date.
TensorFlow 패키지가 지원되는 GPU는 CUdaToolkit 7.0과 CUDNN 6.5 V2를 요구한다. 설치 환경을 위해 cuda 설치 웹 사이트를 방문하길 권하며 그 사이트의 최신 정보를 통해 세부적인 사항을 확인하길 바란다.

The way to reference those devices in TensorFlow is the following one:
TensorFlow에서 장치를 참조하는 방법은 다음과 같다.

“/cpu:0”: To reference the server’s CPU.
“/gpu:0”: The server’s GPU, if only one is available.
“/gpu:1”: The second server’s GPU, and so on.
“/cpu:0”: 서버 CPU 참조
“/gpu:0”: 서버 GPU(1개일 경우)
“/gpu:1”: 서버 GPU(2개일 경우, 숫자는 서버 GPU 개수)

To know in which devices our operations and tensors are assigned we need to create a sesion with the option log_device_placement as True. Let’s see it in the following example:
작동 장치와 tensor가 어디로 할당되었는지 알기 위해서는 세션을 생성해야 하는데 이때 log_device_placement는 참으로 되있어야 한다. 이러한 과정을 다음 예제를 통해 살펴보자.

import tensorflow as tf
a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
c = tf.matmul(a, b)

sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
printsess.run(c)

When the reader tests this code in their computer, a similar output should appear:
자신의 컴퓨터로 위의 코드를 테스트하면 다음과 같은 출력이 나타나야 한다:

. . .
Device mapping:
/job:localhost/replica:0/task:0/gpu:0 -&gt; device: 0, name: Tesla K40c, pci bus id: 0000:08:00.0
. . .
b: /job:localhost/replica:0/task:0/gpu:0
a: /job:localhost/replica:0/task:0/gpu:0
MatMul: /job:localhost/replica:0/task:0/gpu:0
…
[[ 22.28.]
[ 49.64.]]
…

Also, with the result of the operation, it informs to us where is executed each part.
또한 조작의 결과와 함께 각 부분이 실행된 장소를 알 수 있다.

If we want a specific operation to be executed in a specific device, instead of letting the system select automatically a device we can use the variable tf.device to create a device context, so all the operations in that context will have the same device assigned.
시스템에서 자동적으로 장치를 선택하는 것이 아니라 어떤 장치에서 특정한 조작을 실행하길 원한다면 tf.device 변수를 이용해 장치 컨텍스트를 생성하면 된다. 이렇게 하면 컨텍스트의 모든 조작은 동일한 장치에 할당된다.

If we have more that a GPU in the system, the GPU with the lower identifier will be selected by default. In case that we want to execute operations in a different GPU, we have to specify this explicitly. For example, if we want the previous code to be executed in GPU #2 we can use tf.device(‘/gpu:2’) as shown here:
시스템에 여러 GPU가 있을 경우, 낮은 식별자로 된 GPU가 기본(default)으로 선택된다. 다른 GPU에서 조작을 실행하려면 명시적으로 지정해야 한다. 예를 들어, 앞에서 언급된 코드가 GPU #2에서 실행되게 하려면 아래와 같이 tf.device('/gpu:2')를 이용한다. 

import tensorflow as tf

with tf.device('/gpu:2'):
a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
c = tf.matmul(a, b)
sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
printsess.run(c)

Parallelism with several GPUs
여러 GPU에서 병렬화

In case that we have more than one GPU, usually we’d like to use all together to solve the same problem parallelly. For this, we can build our model to distribute the work among several GPUs. We see it in the next example:
한 개 이상의 GPU가 있을 경우 동일한 문제를 병렬적으로 동시에 해결하는 것이 일반적이다. 이를 위해 작업물(work)을 여러 GPU마다 분배하는 모델을 구축할 수 있다. 다음 예제를 통해 살펴보자:

import tensorflow as tf

c = []
for d in ['/gpu:2', '/gpu:3']:
with tf.device(d):
a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3])
b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2])
c.append(tf.matmul(a, b))
with tf.device('/cpu:0'):
sum = tf.add_n(c)

Creates a session with log_device_placement set to True.
log_device_placement를 참으로 하여 세션을 생성한다.

sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
print sess.run(sum)

As we see, the code is equivalent to the previous one but now we have 2 GPUs indicated with tf.device performing a multiplication (both GPUs do the same here, to simplify the example code), and later the CPU performs the addition. Given that we set log_device_placement as true, we can see in the output how the operations are distributed through our devices[45].
볼 수 있듯이 코드는 이전의 것과 동일하지만, 여기서 tf.device로 표시된 2개의 GPU는 곱셈을 수행하고 CPU는 덧셈을 수행한다. log_device_placement가 참으로 설정되어 있으므로 장치마다 조작이 어떻게 분배되었는지 확인할 수 있다. 

. . .
Device mapping:
/job:localhost/replica:0/task:0/gpu:0 -&gt; device: 0, name: Tesla K40c
/job:localhost/replica:0/task:0/gpu:1 -&gt; device: 1, name: Tesla K40c
/job:localhost/replica:0/task:0/gpu:2 -&gt; device: 2, name: Tesla K40c
/job:localhost/replica:0/task:0/gpu:3 -&gt; device: 3, name: Tesla K40c
. . .


. . .

Const_3: /job:localhost/replica:0/task:0/gpu:3
I tensorflow/core/common_runtime/simple_placer.cc:289] Const_3: /job:localhost/replica:0/task:0/gpu:3
Const_2: /job:localhost/replica:0/task:0/gpu:3
I tensorflow/core/common_runtime/simple_placer.cc:289] Const_2: /job:localhost/replica:0/task:0/gpu:3
MatMul_1: /job:localhost/replica:0/task:0/gpu:3
I tensorflow/core/common_runtime/simple_placer.cc:289] MatMul_1: /job:localhost/replica:0/task:0/gpu:3
Const_1: /job:localhost/replica:0/task:0/gpu:2
I tensorflow/core/common_runtime/simple_placer.cc:289] Const_1: /job:localhost/replica:0/task:0/gpu:2
Const: /job:localhost/replica:0/task:0/gpu:2
I tensorflow/core/common_runtime/simple_placer.cc:289] Const: /job:localhost/replica:0/task:0/gpu:2
MatMul: /job:localhost/replica:0/task:0/gpu:2
I tensorflow/core/common_runtime/simple_placer.cc:289] MatMul: /job:localhost/replica:0/task:0/gpu:2
AddN: /job:localhost/replica:0/task:0/cpu:0
I tensorflow/core/common_runtime/simple_placer.cc:289] AddN: /job:localhost/replica:0/task:0/cpu:0
[[44.56.]
[98.128.]]
. . .
 

Code example with GPUs
GPU code 예시

To conclude this brief chapter, we present a snippet of code inspired on the one shared by DamienAymeric in Github[46], computing An+Bn for n=10 comparing the execution time with 1 GPU against 2 GPUs, using the datetime Python package.
본 장을 마무리하기 위해,  Github의 DamienAymeric이 공유한 코드를 제시한다. 이것은 날짜 Python 패키지를 사용하여 1개의 GPU와 2개의 GPU에서 n=10일 때 An+Bn을 계산하는데 걸린 시간을 비교하는 것이다.

First of all, we import the required libraries:
먼저, 필요한 라이브러리를 불러온다:

import numpy as np
import tensorflow as tf
import datetime

We create two matrix with random values, using the numpy package:
numpy 패키지를 이용해 랜덤 값으로 이루어진 두 개의 행렬을 생성한다:

A = np.random.rand(1e4, 1e4).astype('float32')
B = np.random.rand(1e4, 1e4).astype('float32')

n = 10

Then, we create the two structures to store the results:
그리고 나서 결과를 저장할 두 개의 구조(structure)을 생성한다:

c1 = []
c2 = []

Next, we define the matpow() function as follows:
다음으로 matpow() 함수를 다음과 같이 정의한다:

defmatpow(M, n):
    if n &lt; 1: #Abstract cases where n &lt; 1
       return M
    else:
       return tf.matmul(M, matpow(M, n-1))
       
As we’ve seen, to execute the code in a single GPU, we have to specify this as follows:
확인한 것처럼 한 개의 GPU를 실행하기 위해서는 다음과 같이 명시해야 한다:

with tf.device('/gpu:0'):
    a = tf.constant(A)
    b = tf.constant(B)
    c1.append(matpow(a, n))
    c1.append(matpow(b, n))

with tf.device('/cpu:0'):
sum = tf.add_n(c1)

t1_1 = datetime.datetime.now()

with tf.Session(config=tf.ConfigProto(log_device_placement=True)) as sess:
sess.run(sum)
t2_1 = datetime.datetime.now()

And for the case with 2 GPUs, the code is as follows:
GPU가 2개인 경우 코드는 다음과 같다.

with tf.device('/gpu:0'):
    #compute A^n and store result in c2
    a = tf.constant(A)
    c2.append(matpow(a, n))
 
with tf.device('/gpu:1'):
    #compute B^n and store result in c2
    b = tf.constant(B)
    c2.append(matpow(b, n))

with tf.device('/cpu:0'):
    sum = tf.add_n(c2) #Addition of all elements in c2, i.e. A^n + B^n
    t1_2 = datetime.datetime.now()

with tf.Session(config=tf.ConfigProto(log_device_placement=True)) as sess:
    # Runs the op.
    sess.run(sum)
t2_2 = datetime.datetime.now()

Finally we print the results for the registered computation time:
마지막으로 등록된 연산 시간에 대한 결과를 출력한다:

print "Single GPU computation time: " + str(t2_1-t1_1)
print "Multi GPU computation time: " + str(t2_2-t1_2)

Distributed version of TensorFlow

TensonFlow 배포 버전

As I previously said at the beginning of this chapter, on February 2016 Google released the distributed version of TensorFlow, which is supported by gRPC, a high performance open source RPC framework for inter-process communication (the same protocol used by TensorFlow Serving).

앞에서 언급했듯이 2016년 2월 구글은 gRPC가 지원하는 TensorFlow 배포 버전을 발표했는데 여기서 gRPC란 프로세스 간 통신을 위한 고성능 오픈 소스 RPC 프레임워크를 말한다(TensorFlow 서비스에서 사용되는 도일한 프로토콜).

For its usage, the binaries must be built because the package only provides the sources at this time. Given the introductory scope of this book, I will not explain it in the distributed version, but if the reader wants to learn about it, I recommend to start with the oficial site for this distributed version of TensorFlow[47].

TensorFlow 배포 버전을 사용하려면 바이나리를 구축해야 한다. 패키지가 그 시간(바이나리와 관련된 시간)에 한해 소스를 제공하기 때문이다. 이에 대한 자세한 설명은 TensorFlow 배포 버전의 공식 사이트에서 확인할 수 있다. 

As in previous chapters, the code used in this one, can be found in the book’s Github[48]. I hope that this chapter has been illustrative enough to understand how to speedup the code using GPUs.

이전 장과 같이, 배포 버전이 적용된 코드는 본 저서의 Github에서 찾을 수 있다. 이번 장을 통해 GPU를 사용한 코드에서 시간을 단축시키는 방법을 이해할 수 있길 바란다. 

[contents link]
