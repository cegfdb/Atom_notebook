
- [Github ResNeSt](https://github.com/zhanghang1989/ResNeSt)
- [Github bleakie/MaskInsightface](https://github.com/bleakie/MaskInsightface)
- [Group Convolution分组卷积，以及Depthwise Convolution和Global Depthwise Convolution](https://cloud.tencent.com/developer/article/1394912)
- [深度学习中的卷积方式](https://zhuanlan.zhihu.com/p/75972500)
***

# Auto Tuner
## Keras Tuner
  - [Keras Tuner 简介](https://www.tensorflow.org/tutorials/keras/keras_tuner)
    ```py
    import tensorflow as tf
    from tensorflow import keras

    !pip install -q -U keras-tuner
    import kerastuner as kt

    (img_train, label_train), (img_test, label_test) = keras.datasets.fashion_mnist.load_data()
    # Normalize pixel values between 0 and 1
    img_train = img_train.astype('float32') / 255.0
    img_test = img_test.astype('float32') / 255.0

    def model_builder(hp):
        model = keras.Sequential()
        model.add(keras.layers.Flatten(input_shape=(28, 28)))

        # Tune the number of units in the first Dense layer
        # Choose an optimal value between 32-512
        hp_units = hp.Int('units', min_value = 32, max_value = 512, step = 32)
        model.add(keras.layers.Dense(units = hp_units, activation = 'relu'))
        model.add(keras.layers.Dense(10))

        # Tune the learning rate for the optimizer
        # Choose an optimal value from 0.01, 0.001, or 0.0001
        hp_learning_rate = hp.Choice('learning_rate', values = [1e-2, 1e-3, 1e-4])

        model.compile(optimizer = keras.optimizers.Adam(learning_rate = hp_learning_rate),
                      loss = keras.losses.SparseCategoricalCrossentropy(from_logits = True),
                      metrics = ['accuracy'])

        return model

    tuner = kt.Hyperband(model_builder,
                         objective = 'val_accuracy',
                         max_epochs = 10,
                         factor = 3,
                         directory = 'my_dir',
                         project_name = 'intro_to_kt')

    tuner.search(img_train, label_train, epochs = 10, validation_data = (img_test, label_test), callbacks = [ClearTrainingOutput()])

    # Get the optimal hyperparameters
    best_hps = tuner.get_best_hyperparameters(num_trials = 1)[0]

    print(f"""
    The hyperparameter search is complete. The optimal number of units in the first densely-connected
    layer is {best_hps.get('units')} and the optimal learning rate for the optimizer
    is {best_hps.get('learning_rate')}.
    """)

    # Build the model with the optimal hyperparameters and train it on the data
    model = tuner.hypermodel.build(best_hps)
    model.fit(img_train, label_train, epochs = 10, validation_data = (img_test, label_test))
    ```
  - **Tune on cifar10**
    ```py
    import tensorflow as tf
    from tensorflow import keras
    import matplotlib.pyplot as plt
    import kerastuner as kt
    import tensorflow_addons as tfa

    (train_images, train_labels), (test_images, test_labels) = keras.datasets.cifar10.load_data()
    train_images, test_images = train_images / 255.0, test_images / 255.0
    train_labels_oh = tf.one_hot(tf.squeeze(train_labels), depth=10, dtype='uint8')
    test_labels_oh = tf.one_hot(tf.squeeze(test_labels), depth=10, dtype='uint8')
    print(train_images.shape, test_images.shape, train_labels_oh.shape, test_labels_oh.shape)

    def create_model(hp):
        hp_wd = hp.Choice("weight_decay", values=[0.0, 1e-5, 5e-5, 1e-4])
        hp_ls = hp.Choice("label_smoothing", values=[0.0, 0.1])
        hp_dropout = hp.Choice("dropout_rate", values=[0.0, 0.4])

        model = keras.Sequential()
        model.add(keras.layers.Conv2D(32, (3, 3), activation='relu', input_shape=(32, 32, 3)))
        model.add(keras.layers.MaxPooling2D((2, 2)))
        model.add(keras.layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(keras.layers.MaxPooling2D((2, 2)))
        model.add(keras.layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(keras.layers.Flatten())
        model.add(keras.layers.Dense(64, activation='relu'))
        model.add(keras.layers.Dropout(rate=hp_dropout))
        model.add(keras.layers.Dense(10))

        model.compile(
            optimizer=tfa.optimizers.AdamW(weight_decay=hp_wd),
            loss=tf.keras.losses.CategoricalCrossentropy(from_logits=True, label_smoothing=hp_ls),
            metrics = ['accuracy'])

        return model

    tuner = kt.Hyperband(create_model,
                         objective='val_accuracy',
                         max_epochs=50,
                         factor=6,
                         directory='my_dir',
                         project_name='intro_to_kt')

    tuner.search(train_images, train_labels_oh, epochs=50, validation_data=(test_images, test_labels_oh))

    # Get the optimal hyperparameters
    best_hps = tuner.get_best_hyperparameters(num_trials=1)[0]
    print("best parameters: weight_decay = {}, label_smoothing = {}, dropout_rate = {}".format(best_hps.get('weight_decay'), best_hps.get('label_smoothing'), best_hps.get('dropout_rate')))

    # Build the model with the optimal hyperparameters and train it on the data
    model = tuner.hypermodel.build(best_hps)
    model.fit(train_images, train_labels_oh, epochs = 50, validation_data = (test_images, test_labels_oh))
    ```
## TensorBoard HParams
  - [Hyperparameter Tuning with the HParams Dashboard](https://www.tensorflow.org/tensorboard/hyperparameter_tuning_with_hparams)
    ```py
    # Load the TensorBoard notebook extension
    %load_ext tensorboard

    import tensorflow as tf
    from tensorboard.plugins.hparams import api as hp

    fashion_mnist = tf.keras.datasets.fashion_mnist

    (x_train, y_train),(x_test, y_test) = fashion_mnist.load_data()
    x_train, x_test = x_train / 255.0, x_test / 255.0

    HP_NUM_UNITS = hp.HParam('num_units', hp.Discrete([16, 32]))
    HP_DROPOUT = hp.HParam('dropout', hp.RealInterval(0.1, 0.2))
    HP_OPTIMIZER = hp.HParam('optimizer', hp.Discrete(['adam', 'sgd']))

    METRIC_ACCURACY = 'accuracy'

    with tf.summary.create_file_writer('logs/hparam_tuning').as_default():
        hp.hparams_config(
            hparams=[HP_NUM_UNITS, HP_DROPOUT, HP_OPTIMIZER],
            metrics=[hp.Metric(METRIC_ACCURACY, display_name='Accuracy')],
        )

    def train_test_model(hparams):
        model = tf.keras.models.Sequential([
            tf.keras.layers.Flatten(),
            tf.keras.layers.Dense(hparams[HP_NUM_UNITS], activation=tf.nn.relu),
            tf.keras.layers.Dropout(hparams[HP_DROPOUT]),
            tf.keras.layers.Dense(10, activation=tf.nn.softmax),
        ])
        model.compile(
            optimizer=hparams[HP_OPTIMIZER],
            loss='sparse_categorical_crossentropy',
            metrics=['accuracy'],
        )

        # model.fit(
        #   ...,
        #   callbacks=[
        #       tf.keras.callbacks.TensorBoard(logdir),  # log metrics
        #       hp.KerasCallback(logdir, hparams),  # log hparams
        #   ],
        # )
        model.fit(x_train, y_train, epochs=1) # Run with 1 epoch to speed things up for demo purposes
        _, accuracy = model.evaluate(x_test, y_test)
        return accuracy

    def run(run_dir, hparams):
        with tf.summary.create_file_writer(run_dir).as_default():
            hp.hparams(hparams)  # record the values used in this trial
            accuracy = train_test_model(hparams)
            tf.summary.scalar(METRIC_ACCURACY, accuracy, step=1)

    session_num = 0

    for num_units in HP_NUM_UNITS.domain.values:
        for dropout_rate in (HP_DROPOUT.domain.min_value, HP_DROPOUT.domain.max_value):
            for optimizer in HP_OPTIMIZER.domain.values:
                hparams = {
                    HP_NUM_UNITS: num_units,
                    HP_DROPOUT: dropout_rate,
                    HP_OPTIMIZER: optimizer,
                }
                run_name = "run-%d" % session_num
                print('--- Starting trial: %s' % run_name)
                print({h.name: hparams[h] for h in hparams})
                run('logs/hparam_tuning/' + run_name, hparams)
                session_num += 1

    %tensorboard --logdir logs/hparam_tuning
    ```
  - **Tune on cifar10**
    ```py
    %load_ext tensorboard

    import tensorflow as tf
    from tensorboard.plugins.hparams import api as hp
    import tensorflow_addons as tfa

    (train_images, train_labels), (test_images, test_labels) = keras.datasets.cifar10.load_data()
    train_images, test_images = train_images / 255.0, test_images / 255.0
    train_labels_oh = tf.one_hot(tf.squeeze(train_labels), depth=10, dtype='uint8')
    test_labels_oh = tf.one_hot(tf.squeeze(test_labels), depth=10, dtype='uint8')
    print(train_images.shape, test_images.shape, train_labels_oh.shape, test_labels_oh.shape)

    HP_WD = hp.HParam("weight_decay", hp.Discrete([0.0, 1e-5, 5e-5, 1e-4]))
    HP_LS = hp.HParam("label_smoothing", hp.Discrete([0.0, 0.1]))
    HP_DR = hp.HParam("dropout_rate", hp.Discrete([0.0, 0.4]))
    METRIC_ACCURACY = 'accuracy'
    METRIC_LOSS = 'loss'

    with tf.summary.create_file_writer('logs/hparam_tuning_cifar10').as_default():
        hp.hparams_config(
            hparams=[HP_WD, HP_LS, HP_DR],
            metrics=[hp.Metric(METRIC_ACCURACY, display_name='Accuracy'), hp.Metric(METRIC_LOSS, display_name='Loss')],
        )

    def create_model(dropout=1):
        model = keras.models.Sequential()
        model.add(keras.layers.Conv2D(32, (3, 3), activation='relu', input_shape=(32, 32, 3)))
        model.add(keras.layers.MaxPooling2D((2, 2)))
        model.add(keras.layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(keras.layers.MaxPooling2D((2, 2)))
        model.add(keras.layers.Conv2D(64, (3, 3), activation='relu'))
        model.add(keras.layers.Flatten())
        model.add(keras.layers.Dense(64, activation='relu'))
        if dropout > 0 and dropout < 1:
            model.add(keras.layers.Dropout(dropout))
        model.add(keras.layers.Dense(10))
        return model

    def train_test_model(hparams, epochs=1):
        model = create_model(hparams[HP_DR])
        model.compile(
            optimizer=tfa.optimizers.AdamW(weight_decay=hparams[HP_WD]),
            loss=tf.keras.losses.CategoricalCrossentropy(label_smoothing=hparams[HP_LS], from_logits=True),
            metrics=['accuracy'],
        )

        # model.fit(
        #   ...,
        #   callbacks=[
        #       tf.keras.callbacks.TensorBoard(logdir),  # log metrics
        #       hp.KerasCallback(logdir, hparams),  # log hparams
        #   ],
        # )
        hist = model.fit(train_images, train_labels_oh, epochs=epochs, validation_data=(test_images, test_labels_oh)) # Run with 1 epoch to speed things up for demo purposes
        return max(hist.history["val_accuracy"]), min(hist.history["val_loss"])

    def run(run_dir, hparams):
        with tf.summary.create_file_writer(run_dir).as_default():
            hp.hparams(hparams)  # record the values used in this trial
            val_accuracy, val_loss = train_test_model(hparams, epochs=20)
            tf.summary.scalar(METRIC_ACCURACY, val_accuracy, step=1)
            tf.summary.scalar(METRIC_LOSS, val_loss, step=1)

    session_num = 0
    for dr in HP_DR.domain.values:
        for label_smoothing in HP_LS.domain.values:
            for wd in HP_WD.domain.values:
                hparams = {
                    HP_WD: wd,
                    HP_LS: label_smoothing,
                    HP_DR: dr,
                }
                run_name = "run-%d" % session_num
                print('--- Starting trial: %s' % run_name)
                print({h.name: hparams[h] for h in hparams})
                run('logs/hparam_tuning_cifar10/' + run_name, hparams)
                session_num += 1

    %tensorboard --logdir logs/hparam_tuning_cifar10
    ```
***

# TfLite
## TFLite Model Benchmark Tool
  - [TFLite Model Benchmark Tool](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/lite/tools/benchmark)
    ```sh
    cd ~/workspace/tensorflow.arm32
    ./configure
    bazel build -c opt --config=android_arm tensorflow/lite/tools/benchmark:benchmark_model
    bazel build --config opt --config monolithic --define tflite_with_xnnpack=false tensorflow/lite/tools/benchmark:benchmark_model

    adb push bazel-bin/tensorflow/lite/tools/benchmark/benchmark_model /data/local/tmp
    adb shell chmod +x /data/local/tmp/benchmark_model

    cd ~/workspace/examples/lite/examples/image_classification/android/app/src/main/assets
    adb push mobilenet_v1_1.0_224_quant.tflite /data/local/tmp
    adb push mobilenet_v1_1.0_224.tflite /data/local/tmp
    adb push efficientnet-lite0-int8.tflite /data/local/tmp
    adb push efficientnet-lite0-fp32.tflite /data/local/tmp
    ```
  - **参数**
    - **--graph** 字符串，TFLite 模型路径
    - **--enable_op_profiling** true / false，是否测试每个步骤的执行时间: bool (default=false) Whether to enable per-operator profiling measurement.
    - **--nnum_threads** 整数值，线程数量
    - **--use_gpu** true / false，是否使用 GPU
    - **--use_nnapi** true / false，是否使用 nnapi
    - **--use_xnnpack** true / false，是否使用 xnnpack
    - **--use_coreml** true / false，是否使用 coreml
  - **Int8 模型 nnapi 测试**
    ```cpp
    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/mobilenet_v1_1.0_224_quant.tflite --num_threads=1
    INFO: Initialized TensorFlow Lite runtime.
    The input model file size (MB): 4.27635
    Inference timings in us: Init: 5388, First inference: 101726, Warmup (avg): 92755.2, Inference (avg): 90865.9

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/mobilenet_v1_1.0_224_quant.tflite --num_threads=4
    INFO: Initialized TensorFlow Lite runtime.
    The input model file size (MB): 4.27635
    Inference timings in us: Init: 5220, First inference: 50829, Warmup (avg): 29745.6, Inference (avg): 27044.7

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/mobilenet_v1_1.0_224_quant.tflite --num_threads=1 --use_nnapi=true
    Explicitly applied NNAPI delegate, and the model graph will be completely executed by the delegate.
    The input model file size (MB): 4.27635
    Inference timings in us: Init: 25558, First inference: 9992420, Warmup (avg): 9.99242e+06, Inference (avg): 8459.69

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/mobilenet_v1_1.0_224_quant.tflite --num_threads=4 --use_nnapi=true
    Explicitly applied NNAPI delegate, and the model graph will be completely executed by the delegate.
    The input model file size (MB): 4.27635
    Inference timings in us: Init: 135723, First inference: 10013451, Warmup (avg): 1.00135e+07, Inference (avg): 8413.35

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/efficientnet-lite0-int8.tflite --num_threads=1
    INFO: Initialized TensorFlow Lite runtime.
    The input model file size (MB): 5.42276
    Inference timings in us: Init: 16296, First inference: 111237, Warmup (avg): 100603, Inference (avg): 98068.9

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/efficientnet-lite0-int8.tflite --num_threads=4
    INFO: Initialized TensorFlow Lite runtime.
    The input model file size (MB): 5.42276
    Inference timings in us: Init: 13910, First inference: 52150, Warmup (avg): 30097.1, Inference (avg): 28823.8

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/efficientnet-lite0-int8.tflite --num_threads=1 --use_nnapi=true
    Explicitly applied NNAPI delegate, and the model graph will be partially executed by the delegate w/ 11 delegate kernels.
    The input model file size (MB): 5.42276
    Inference timings in us: Init: 30724, First inference: 226753, Warmup (avg): 171396, Inference (avg): 143630

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/efficientnet-lite0-int8.tflite --num_threads=4 --use_nnapi=true
    Explicitly applied NNAPI delegate, and the model graph will be partially executed by the delegate w/ 11 delegate kernels.
    The input model file size (MB): 5.42276
    Inference timings in us: Init: 32209, First inference: 207213, Warmup (avg): 75055, Inference (avg): 53974.5
    ```
  - **Float32 模型 xnnpack 测试** 不经过量化的浮点型模型可以使用 `xnnpack` 加速
    ```cpp
    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/mobilenet_v1_1.0_224.tflite --num_threads=1
    INFO: Initialized TensorFlow Lite runtime.
    The input model file size (MB): 16.9008
    Inference timings in us: Init: 2491, First inference: 183222, Warmup (avg): 170631, Inference (avg): 163455

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/mobilenet_v1_1.0_224.tflite --num_threads=4
    INFO: Initialized TensorFlow Lite runtime.
    The input model file size (MB): 16.9008
    Inference timings in us: Init: 2482, First inference: 101750, Warmup (avg): 58520.2, Inference (avg): 52692.1

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/mobilenet_v1_1.0_224.tflite --num_threads=1 --use_xnnpack=true
    Explicitly applied XNNPACK delegate, and the model graph will be partially executed by the delegate w/ 2 delegate kernels.
    The input model file size (MB): 16.9008
    Inference timings in us: Init: 55797, First inference: 167033, Warmup (avg): 160670, Inference (avg): 159191

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/mobilenet_v1_1.0_224.tflite --num_threads=4 --use_xnnpack=true
    Explicitly applied XNNPACK delegate, and the model graph will be partially executed by the delegate w/ 2 delegate kernels.
    The input model file size (MB): 16.9008
    Inference timings in us: Init: 61780, First inference: 75098, Warmup (avg): 51450.6, Inference (avg): 47564.3

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/efficientnet-lite0-fp32.tflite --num_threads=1
    INFO: Initialized TensorFlow Lite runtime.
    The input model file size (MB): 18.5702
    Inference timings in us: Init: 6697, First inference: 169388, Warmup (avg): 148210, Inference (avg): 141517

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/efficientnet-lite0-fp32.tflite --num_threads=4
    INFO: Initialized TensorFlow Lite runtime.
    The input model file size (MB): 18.5702
    Inference timings in us: Init: 4137, First inference: 84115, Warmup (avg): 52832.1, Inference (avg): 52848.9

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/efficientnet-lite0-fp32.tflite --num_threads=1 --use_xnnpack=true
    Explicitly applied XNNPACK delegate, and the model graph will be completely executed by the delegate.
    The input model file size (MB): 18.5702
    Inference timings in us: Init: 53629, First inference: 120858, Warmup (avg): 114820, Inference (avg): 112744

    $ adb shell /data/local/tmp/benchmark_model --graph=/data/local/tmp/efficientnet-lite0-fp32.tflite --num_threads=4 --use_xnnpack=true
    Explicitly applied XNNPACK delegate, and the model graph will be completely executed by the delegate.
    The input model file size (MB): 18.5702
    Inference timings in us: Init: 52265, First inference: 45786, Warmup (avg): 42789.7, Inference (avg): 40117.3
    ```
***

# Quantization aware training in Keras example
  - [Quantization aware training in Keras example](https://www.tensorflow.org/model_optimization/guide/quantization/training_example)
  - **Conda install `tf-nightly` and `tensorflow-model-optimization`**
    ```sh
    conda create -n tf-nightly
    conda activate tf-nightly
    pip install tf-nightly glob2 pandas tqdm scikit-image scikit-learn ipython
    pip install -q tensorflow-model-optimization

    # Install cuda 10.1 if not installed
    conda install cudnn=7.6.5=cuda10.1_0

    source activate tf-nightly
    ```
  - **Train a MNIST model**
    ```py
    import os
    import tensorflow as tf
    from tensorflow import keras

    # Load MNIST dataset
    mnist = keras.datasets.mnist
    (train_images, train_labels), (test_images, test_labels) = mnist.load_data()

    # Normalize the input image so that each pixel value is between 0 to 1.
    train_images = train_images / 255.0
    test_images = test_images / 255.0

    # Define the model architecture.
    model = keras.Sequential([
      keras.layers.InputLayer(input_shape=(28, 28)),
      keras.layers.Reshape(target_shape=(28, 28, 1)),
      keras.layers.Conv2D(filters=12, kernel_size=(3, 3), activation='relu'),
      keras.layers.MaxPooling2D(pool_size=(2, 2)),
      keras.layers.Flatten(),
      keras.layers.Dense(10)
    ])

    # Train the digit classification model
    model.compile(optimizer='adam',
                  loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
                  metrics=['accuracy'])

    model.fit(train_images, train_labels, epochs=1, validation_data=(test_images, test_labels))
    # 1875/1875 [==============================] - 5s 3ms/step - loss: 0.5267 - accuracy: 0.8500 - val_loss: 0.1736 - val_accuracy: 0.9523
    ```
  - **Clone and fine-tune pre-trained model with quantization aware training**
    - 将 `quantization aware` 应用到整个模型，模型额的每一层将以 `quant` 开头
    - 训练后的模型是 `quantization aware` 的，但还没有被量化
    ```py
    import tensorflow_model_optimization as tfmot
    quantize_model = tfmot.quantization.keras.quantize_model

    # q_aware stands for for quantization aware.
    q_aware_model = quantize_model(model)
    # `quantize_model` requires a recompile.
    q_aware_model.compile(optimizer='adam',
                  loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
                  metrics=['accuracy'])

    q_aware_model.summary()
    print([ii.name for ii in q_aware_model.layers])
    # ['quantize_layer', 'quant_reshape', 'quant_conv2d', 'quant_max_pooling2d', 'quant_flatten', 'quant_dense']
    ```
  - **Train and evaluate the model against baseline** 作为对比，在一个小的数据集上做 fine tune
    ```py
    train_images_subset = train_images[0:1000] # out of 60000
    train_labels_subset = train_labels[0:1000]
    q_aware_model.fit(train_images_subset, train_labels_subset, batch_size=500, epochs=1, validation_data=(test_images, test_labels))
    # 2/2 [==============================] - 1s 325ms/step - loss: 0.1783 - accuracy: 0.9447 - val_loss: 0.1669 - val_accuracy: 0.9551
    ```
    **Evaluate**
    ```py
    _, baseline_model_accuracy = model.evaluate(test_images, test_labels, verbose=0)
    print('Baseline test accuracy:', baseline_model_accuracy)
    # Baseline test accuracy: 0.9523000121116638

    _, q_aware_model_accuracy = q_aware_model.evaluate(test_images, test_labels, verbose=0)
    print('Quant test accuracy:', q_aware_model_accuracy)
    # Quant test accuracy: 0.9550999999046326
    ```
  - **More training**
    ```py
    model.fit(train_images, train_labels, epochs=5, validation_data=(test_images, test_labels))
    # Epoch 1/5 1875/1875 - 3s 2ms/step - loss: 0.1368 - accuracy: 0.9613 - val_loss: 0.1023 - val_accuracy: 0.9706
    # Epoch 2/5 1875/1875 - 3s 2ms/step - loss: 0.0921 - accuracy: 0.9742 - val_loss: 0.0810 - val_accuracy: 0.9763
    # Epoch 3/5 1875/1875 - 3s 2ms/step - loss: 0.0732 - accuracy: 0.9786 - val_loss: 0.0754 - val_accuracy: 0.9746
    # Epoch 4/5 1875/1875 - 3s 1ms/step - loss: 0.0624 - accuracy: 0.9821 - val_loss: 0.0709 - val_accuracy: 0.9768
    # Epoch 5/5 1875/1875 - 3s 1ms/step - loss: 0.0550 - accuracy: 0.9841 - val_loss: 0.0658 - val_accuracy: 0.9776

    q_aware_model.fit(train_images, train_labels, epochs=5, validation_data=(test_images, test_labels))
    # Epoch 1/5 1875/1875 - 6s 3ms/step - loss: 0.1359 - accuracy: 0.9619 - val_loss: 0.0974 - val_accuracy: 0.9720
    # Epoch 2/5 1875/1875 - 6s 3ms/step - loss: 0.0895 - accuracy: 0.9746 - val_loss: 0.0779 - val_accuracy: 0.9768
    # Epoch 3/5 1875/1875 - 6s 3ms/step - loss: 0.0708 - accuracy: 0.9795 - val_loss: 0.0650 - val_accuracy: 0.9790
    # Epoch 4/5 1875/1875 - 6s 3ms/step - loss: 0.0605 - accuracy: 0.9821 - val_loss: 0.0636 - val_accuracy: 0.9798
    # Epoch 5/5 1875/1875 - 6s 3ms/step - loss: 0.0541 - accuracy: 0.9840 - val_loss: 0.0666 - val_accuracy: 0.9802
    ```
  - **Create quantized model for TFLite backend** 创建量化模型
    ```py
    converter = tf.lite.TFLiteConverter.from_keras_model(q_aware_model)
    converter.optimizations = [tf.lite.Optimize.DEFAULT]

    quantized_tflite_model = converter.convert()
    ```
    ```py
    mnist = keras.datasets.mnist
    (train_images, train_labels), (test_images, test_labels) = mnist.load_data()
    train_images, test_images = train_images.astype(np.float32) / 255.0, test_images.astype(np.float32) / 255.0
    def representative_data_gen():
        for input_value in tf.data.Dataset.from_tensor_slices(train_images).batch(1).take(100):
            # Model has only one input so each data point has one element.
            yield [input_value]

    converter = tf.lite.TFLiteConverter.from_keras_model(q_aware_model)
    converter.optimizations = [tf.lite.Optimize.DEFAULT]
    converter.representative_dataset = representative_data_gen
    # Ensure that if any ops can't be quantized, the converter throws an error
    converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
    # Set the input and output tensors to uint8 (APIs added in r2.3)
    converter.inference_input_type = tf.uint8
    converter.inference_output_type = tf.uint8

    tflite_model_quant = converter.convert()

    interpreter = tf.lite.Interpreter(model_content=tflite_model_quant)
    input_type = interpreter.get_input_details()[0]['dtype']
    print('input: ', input_type)
    output_type = interpreter.get_output_details()[0]['dtype']
    print('output: ', output_type)
    ```
    **Evaluate**
    ```py
    import numpy as np

    def evaluate_model(interpreter):
        input_index = interpreter.get_input_details()[0]["index"]
        output_index = interpreter.get_output_details()[0]["index"]
        prediction_digits = []
        for i, test_image in enumerate(test_images):
            test_image = np.expand_dims(test_image, axis=0).astype(np.float32)
            interpreter.set_tensor(input_index, test_image)
            interpreter.invoke()
            output = interpreter.tensor(output_index)
            digit = np.argmax(output()[0])
            prediction_digits.append(digit)
        prediction_digits = np.array(prediction_digits)
        accuracy = (prediction_digits == test_labels).mean()
        return accuracy

    interpreter = tf.lite.Interpreter(model_content=quantized_tflite_model)
    interpreter.allocate_tensors()
    print('Quant TFLite test accuracy:', evaluate_model(interpreter))
    # Quant TFLite test accuracy: 0.9826

    _, q_aware_model_accuracy = q_aware_model.evaluate(test_images, test_labels, verbose=0)
    print('Quant TF test accuracy:', q_aware_model_accuracy)
    # Quant TF test accuracy: 0.982699990272522
    ```
  - **全整型量化 [ Fail ??? ]** `RuntimeError: Quantization not yet supported for op: 'DEQUANTIZE'.`
    ```py
    import tensorflow as tf
    from tensorflow import keras
    import tensorflow_model_optimization as tfmot

    # Load MNIST dataset
    (train_images, train_labels), (test_images, test_labels) = keras.datasets.mnist.load_data()
    train_images, test_images = train_images / 255.0, test_images / 255.0

    # Define the model architecture.
    model = keras.Sequential([
        keras.layers.InputLayer(input_shape=(28, 28)),
        keras.layers.Flatten(),
        keras.layers.Dense(10)
    ])

    # Train the digit classification model
    model.compile(optimizer='adam', loss=keras.losses.SparseCategoricalCrossentropy(from_logits=True), metrics=['accuracy'])
    model.fit(train_images, train_labels, epochs=1, validation_data=(test_images, test_labels))
    # 1875/1875 [==============================] - 2s 946us/step - loss: 0.7303 - accuracy: 0.8100 - val_loss: 0.3097 - val_accuracy: 0.9117

    # Train the quantization aware model
    q_aware_model = tfmot.quantization.keras.quantize_model(model)
    q_aware_model.compile(optimizer='adam', loss=keras.losses.SparseCategoricalCrossentropy(from_logits=True), metrics=['accuracy'])
    q_aware_model.fit(train_images, train_labels, epochs=1, validation_data=(test_images, test_labels))
    # 1875/1875 [==============================] - 2s 1ms/step - loss: 0.3107 - accuracy: 0.9136 - val_loss: 0.2824 - val_accuracy: 0.9225

    # Define the representative data.
    def representative_data_gen():
        for input_value in tf.data.Dataset.from_tensor_slices(train_images.astype("float32")).batch(1).take(100):
            yield [input_value]

    # Successful converting from model
    converter = tf.lite.TFLiteConverter.from_keras_model(model)
    converter.optimizations = [tf.lite.Optimize.DEFAULT]
    converter.representative_dataset = representative_data_gen
    tflite_model = converter.convert()

    # Successful converting from model to uint8
    converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
    converter.inference_input_type = tf.uint8
    converter.inference_output_type = tf.uint8
    tflite_model_quant = converter.convert()

    # Successful converting from q_aware_model
    q_converter = tf.lite.TFLiteConverter.from_keras_model(q_aware_model)
    q_converter.optimizations = [tf.lite.Optimize.DEFAULT]
    q_converter.representative_dataset = representative_data_gen
    q_tflite_model = q_converter.convert()

    # Fail converting from q_aware_model to uint8
    q_converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
    q_converter.inference_input_type = tf.uint8
    q_converter.inference_output_type = tf.uint8
    q_tflite_model_quant = q_converter.convert()
    # RuntimeError: Quantization not yet supported for op: 'DEQUANTIZE'.

    # Successful converting from model to uint8
    converter = tf.lite.TFLiteConverter.from_keras_model(model)
    converter.optimizations = [tf.lite.Optimize.DEFAULT]
    converter.representative_dataset = representative_data_gen
    converter.inference_input_type = tf.uint8
    converter.inference_output_type = tf.uint8
    tflite_model_quant = converter.convert()

    # Fail converting from q_aware_model to uint8
    q_converter = tf.lite.TFLiteConverter.from_keras_model(q_aware_model)
    q_converter.optimizations = [tf.lite.Optimize.DEFAULT]
    q_converter.representative_dataset = representative_data_gen
    q_converter.inference_input_type = tf.uint8
    q_converter.inference_output_type = tf.uint8
    q_tflite_model_quant = q_converter.convert()
    # RuntimeError: Unsupported output type UINT8 for output tensor 'Identity' of type FLOAT32.

    interpreter = tf.lite.Interpreter(model_content=tflite_model_quant)
    print('input: ', interpreter.get_input_details()[0]['dtype'])
    print('output: ', interpreter.get_output_details()[0]['dtype'])
    ```
***

# Replace UpSampling2D with Conv2DTranspose
## Conv2DTranspose output shape
  ```py
  for strides in range(1, 4):
      for kernel_size in range(1, 4):
          aa = keras.layers.Conv2DTranspose(3, kernel_size, padding='same', strides=strides)
          aa.build([1, 3, 3, 3])
          print("[SAME] kernel_size: {}, strides: {}, shape: {}".format(kernel_size, strides, aa(tf.ones([1, 3, 3, 3], dtype='float32')).shape.as_list()))
  # [SAME] kernel_size: 1, strides: 1, shape: [1, 3, 3, 3]
  # [SAME] kernel_size: 2, strides: 1, shape: [1, 3, 3, 3]
  # [SAME] kernel_size: 3, strides: 1, shape: [1, 3, 3, 3]
  # [SAME] kernel_size: 1, strides: 2, shape: [1, 6, 6, 3]
  # [SAME] kernel_size: 2, strides: 2, shape: [1, 6, 6, 3]
  # [SAME] kernel_size: 3, strides: 2, shape: [1, 6, 6, 3]
  # [SAME] kernel_size: 1, strides: 3, shape: [1, 9, 9, 3]
  # [SAME] kernel_size: 2, strides: 3, shape: [1, 9, 9, 3]
  # [SAME] kernel_size: 3, strides: 3, shape: [1, 9, 9, 3]

  for strides in range(1, 4):
      for kernel_size in range(1, 5):
          aa = keras.layers.Conv2DTranspose(3, kernel_size, padding='valid', strides=strides)
          aa.build([1, 3, 3, 3])
          print("[VALID] kernel_size: {}, strides: {}, shape: {}".format(kernel_size, strides, aa(tf.ones([1, 3, 3, 3], dtype='float32')).shape.as_list()))
  # [VALID] kernel_size: 1, strides: 1, shape: [1, 3, 3, 3]
  # [VALID] kernel_size: 2, strides: 1, shape: [1, 4, 4, 3]
  # [VALID] kernel_size: 3, strides: 1, shape: [1, 5, 5, 3]
  # [VALID] kernel_size: 4, strides: 1, shape: [1, 6, 6, 3]
  # [VALID] kernel_size: 1, strides: 2, shape: [1, 6, 6, 3]
  # [VALID] kernel_size: 2, strides: 2, shape: [1, 6, 6, 3]
  # [VALID] kernel_size: 3, strides: 2, shape: [1, 7, 7, 3]
  # [VALID] kernel_size: 4, strides: 2, shape: [1, 8, 8, 3]
  # [VALID] kernel_size: 1, strides: 3, shape: [1, 9, 9, 3]
  # [VALID] kernel_size: 2, strides: 3, shape: [1, 9, 9, 3]
  # [VALID] kernel_size: 3, strides: 3, shape: [1, 9, 9, 3]
  # [VALID] kernel_size: 4, strides: 3, shape: [1, 10, 10, 3]
  ```
## Nearest interpolation
  - **Image methods**
    ```py
    imsize = 3
    x, y = np.ogrid[:imsize, :imsize]
    img = np.repeat((x + y)[..., np.newaxis], 3, 2) / float(imsize + imsize)
    plt.imshow(img, interpolation='none')

    import tensorflow.keras.backend as K
    iaa = tf.image.resize(img, (6, 6), method='nearest')
    ibb = K.resize_images(tf.expand_dims(tf.cast(img, 'float32'), 0), 2, 2, K.image_data_format(), interpolation='nearest')
    ```
  - **UpSampling2D**
    ```py
    aa = keras.layers.UpSampling2D((2, 2), interpolation='nearest')
    icc = aa(tf.expand_dims(tf.cast(img, 'float32'), 0)).numpy()[0]

    print(np.allclose(iaa, icc))
    # True
    ```
  - **tf.nn.conv2d_transpose**
    ```py
    def nearest_upsample_weights(factor, number_of_classes=3):
        filter_size = 2 * factor - factor % 2
        weights = np.zeros((filter_size, filter_size, number_of_classes, number_of_classes), dtype=np.float32)
        upsample_kernel = np.zeros([filter_size, filter_size])
        upsample_kernel[1:factor + 1, 1:factor + 1] = 1

        for i in range(number_of_classes):
            weights[:, :, i, i] = upsample_kernel
        return weights

    channel, factor = 3, 2
    idd = tf.nn.conv2d_transpose(tf.expand_dims(tf.cast(img, 'float32'), 0), nearest_upsample_weights(factor, channel), output_shape=[1, img.shape[0] * factor, img.shape[1] * factor, channel], strides=factor, padding='SAME')
    print(np.allclose(iaa, idd))
    # True

    # Output shape can be different values
    channel, factor = 3, 3
    print(tf.nn.conv2d_transpose(tf.expand_dims(tf.cast(img, 'float32'), 0), nearest_upsample_weights(factor, channel), output_shape=[1, img.shape[0] * factor, img.shape[1] * factor, channel], strides=factor, padding='SAME').shape)
    # (1, 9, 9, 3)
    print(tf.nn.conv2d_transpose(tf.expand_dims(tf.cast(img, 'float32'), 0), nearest_upsample_weights(factor, channel), output_shape=[1, img.shape[0] * factor - 1, img.shape[1] * factor - 1, channel], strides=factor, padding='SAME').shape)
    # (1, 8, 8, 3)
    print(tf.nn.conv2d_transpose(tf.expand_dims(tf.cast(img, 'float32'), 0), nearest_upsample_weights(factor, channel), output_shape=[1, img.shape[0] * factor - 2, img.shape[1] * factor - 2, channel], strides=factor, padding='SAME').shape)
    # (1, 7, 7, 3)
    ```
  - **Conv2DTranspose**
    ```py
    bb = keras.layers.Conv2DTranspose(channel, 2 * factor - factor % 2, padding='same', strides=factor, use_bias=False)
    bb.build([None, None, None, channel])
    bb.set_weights([nearest_upsample_weights(factor, channel)])
    iee = bb(tf.expand_dims(img.astype('float32'), 0)).numpy()[0]
    print(np.allclose(iaa, iee))
    # True
    ```
## Bilinear
  - [pytorch_bilinear_conv_transpose.py](https://gist.github.com/mjstevens777/9d6771c45f444843f9e3dce6a401b183)
  - [Upsampling and Image Segmentation with Tensorflow and TF-Slim](http://warmspringwinds.github.io/tensorflow/tf-slim/2016/11/22/upsampling-and-image-segmentation-with-tensorflow-and-tf-slim/)
  - **UpSampling2D**
    ```py
    imsize = 3
    x, y = np.ogrid[:imsize, :imsize]
    img = np.repeat((x + y)[..., np.newaxis], 3, 2) / float(imsize + imsize)
    plt.imshow(img, interpolation='none')

    channel, factor = 3, 3
    iaa = tf.image.resize(img, (img.shape[0] * factor, img.shape[1] * factor), method='bilinear')

    aa = keras.layers.UpSampling2D((factor, factor), interpolation='bilinear')
    ibb = aa(tf.expand_dims(tf.cast(img, 'float32'), 0)).numpy()[0]
    print(np.allclose(iaa, ibb))
    # True
    ```
  - **Pytorch BilinearConvTranspose2d**
    ```py
    import torch
    import torch.nn as nn

    class BilinearConvTranspose2d(nn.ConvTranspose2d):
        def __init__(self, channels, stride, groups=1):
            if isinstance(stride, int):
                stride = (stride, stride)

            kernel_size = (2 * stride[0] - stride[0] % 2, 2 * stride[1] - stride[1] % 2)
            # padding = (stride[0] - 1, stride[1] - 1)
            padding = 1
            super().__init__(channels, channels, kernel_size=kernel_size, stride=stride, padding=padding, groups=groups)

        def reset_parameters(self):
            nn.init.constant(self.bias, 0)
            nn.init.constant(self.weight, 0)
            bilinear_kernel = self.bilinear_kernel(self.stride)
            for i in range(self.in_channels):
                j = i if self.groups == 1 else 0
                self.weight.data[i, j] = bilinear_kernel

        @staticmethod
        def bilinear_kernel(stride):
            num_dims = len(stride)

            shape = (1,) * num_dims
            bilinear_kernel = torch.ones(*shape)

            # The bilinear kernel is separable in its spatial dimensions
            # Build up the kernel channel by channel
            for channel in range(num_dims):
                channel_stride = stride[channel]
                kernel_size = 2 * channel_stride - channel_stride % 2
                # e.g. with stride = 4
                # delta = [-3, -2, -1, 0, 1, 2, 3]
                # channel_filter = [0.25, 0.5, 0.75, 1.0, 0.75, 0.5, 0.25]
                # delta = torch.arange(1 - channel_stride, channel_stride)
                delta = torch.arange(0, kernel_size)
                delta = delta - (channel_stride - 0.5) if channel_stride % 2 == 0 else delta - (channel_stride - 1)
                channel_filter = (1 - torch.abs(delta / float(channel_stride)))
                # Apply the channel filter to the current channel
                shape = [1] * num_dims
                shape[channel] = kernel_size
                bilinear_kernel = bilinear_kernel * channel_filter.view(shape)
            return bilinear_kernel

    aa = BilinearConvTranspose2d(channel, factor)
    cc = aa(torch.from_numpy(np.expand_dims(img.transpose(2, 0, 1), 0).astype('float32')))
    icc = cc.detach().numpy()[0].transpose(1, 2, 0)
    print(np.allclose(iaa, icc))
    # False
    ```
  - **tf.nn.conv2d_transpose**
    ```py
    # This is same with pytorch bilinear kernel
    def upsample_filt(size):
        factor = (size + 1) // 2
        center = factor - 1 if size % 2 == 1 else factor - 0.5
        og = np.ogrid[:size, :size]
        return (1 - abs(og[0] - center) / factor) * (1 - abs(og[1] - center) / factor)

    def bilinear_upsample_weights(factor, number_of_classes=3):
        filter_size = 2 * factor - factor % 2
        weights = np.zeros((filter_size, filter_size, number_of_classes, number_of_classes), dtype=np.float32)
        upsample_kernel = upsample_filt(filter_size)

        for i in range(number_of_classes):
            weights[:, :, i, i] = upsample_kernel
        return weights

    idd = tf.nn.conv2d_transpose(tf.expand_dims(tf.cast(img, 'float32'), 0), bilinear_upsample_weights(factor, channel), output_shape=[1, img.shape[0] * factor, img.shape[1] * factor, channel], strides=factor, padding='SAME')[0]
    print(np.allclose(icc, idd))
    # True
    ```
  - **Conv2DTranspose**
    ```py
    aa = keras.layers.Conv2DTranspose(channel, 2 * factor - factor % 2, padding='same', strides=factor, use_bias=False)
    aa.build([None, None, None, channel])
    aa.set_weights([bilinear_upsample_weights(factor, channel)])
    iee = aa(tf.expand_dims(tf.cast(img, 'float32'), 0)).numpy()[0]
    ```
  - **Plot**
    ```py
    fig, axes = plt.subplots(1, 6, figsize=(18, 3))
    imgs = [img, iaa, ibb, icc, idd, iee]
    names = ["Orignal", "tf.image.resize", "UpSampling2D", "Pytorch ConvTranspose2d", "tf.nn.conv2d_transpose", "TF Conv2DTranspose"]
    for ax, imm, nn in zip(axes, imgs, names):
        ax.imshow(imm)
        ax.axis('off')
        ax.set_title(nn)
    plt.tight_layout()
    ```
    ```py
    new_rows = ((rows - 1) * strides[0] + kernel_size[0] - 2 * padding[0] + output_padding[0])
    new_cols = ((cols - 1) * strides[1] + kernel_size[1] - 2 * padding[1] + output_padding[1])
    ```
## Clone model
  ```py
  def convert_UpSampling2D_layer(layer):
      print(layer.name)
      if isinstance(layer, keras.layers.UpSampling2D):
          print(">>>> Convert UpSampling2D <<<<")
          channel = layer.input.shape[-1]
          factor = 2
          aa = keras.layers.Conv2DTranspose(channel, 2 * factor - factor % 2, padding='same', strides=factor, use_bias=False)
          aa.build(layer.input.shape)
          aa.set_weights([bilinear_upsample_weights(factor, number_of_classes=channel)])
          return aa
      return layer

  mm = keras.models.load_model('aa.h5', compile=False)
  mmn = keras.models.clone_model(mm, clone_function=convert_UpSampling2D_layer)
  ```
***

# Replace PReLU with DepthwiseConv2D
  ```py
  '''
  mm = keras.models.load_model('./checkpoints/keras_se_mobile_facenet_emore_VIII_basic_agedb_30_epoch_12_0.931000.h5')
  aa = mm.layers[-7]
  ii = np.arange(-1, 1, 2 / (7 * 7 * 512), dtype=np.float32)[:7 * 7 * 512].reshape([1, 7, 7, 512])
  ee = my_activate_test(ii, weights=aa.get_weights())
  np.alltrue(aa(ii) == ee)
  '''
  def my_activate_test(inputs, weights=None):
      channel_axis = 1 if K.image_data_format() == "channels_first" else -1
      pos = K.relu(inputs)
      nn = DepthwiseConv2D((1, 1), depth_multiplier=1, use_bias=False)
      if weights is not None:
          nn.build(inputs.shape)
          nn.set_weights([tf.reshape(weights[id], nn.weights[id].shape) for id, ii in enumerate(weights)])
      neg = -1 * nn(K.relu(-1 * inputs))
      return pos + neg
  ```
  ```py
  from backbones import mobile_facenet_mnn
  bb = mobile_facenet_mnn.mobile_facenet(256, (112, 112, 3), 0.4, use_se=True)
  bb.build((112, 112, 3))

  bb_id = 0
  for id, ii in enumerate(mm.layers):
      print(id, ii.name)
      if isinstance(ii, keras.layers.PReLU):
          print("PReLU")
          nn = bb.layers[bb_id + 2]
          print(bb_id, nn.name)
          nn.set_weights([tf.reshape(wii, nn.weights[wid].shape) for wid, wii in enumerate(ii.get_weights())])
          bb_id += 6
      else:
          nn = bb.layers[bb_id]
          print(bb_id, nn.name)
          nn.set_weights(ii.get_weights())
          bb_id += 1

  inputs = bb.inputs[0]
  embedding = bb.outputs[0]
  output = keras.layers.Dense(tt.classes, name=tt.softmax, activation="softmax")(embedding)
  model = keras.models.Model(inputs, output)
  model.layers[-1].set_weights(tt.model.layers[-2].get_weights())
  model_c = keras.models.Model(model.inputs[0], keras.layers.concatenate([bb.outputs[0], model.outputs[-1]]))
  model_c.compile(optimizer=tt.model.optimizer, loss=tt.model.loss, metrics=tt.model.metrics)
  model_c.optimizer.set_weights(tt.model.optimizer.get_weights())
  ```
  **keras.models.clone_model**
  ```py
  from tensorflow.keras import backend as K
  from tensorflow.keras.layers import DepthwiseConv2D

  # MUST be a customized layer
  # Using DepthwiseConv2D re-implementing PReLU, as MNN doesnt support it...
  class My_PRELU_act(keras.layers.Layer):
      def __init__(self, **kwargs):
          super(My_PRELU_act, self).__init__(**kwargs)
          # channel_axis = 1 if K.image_data_format() == "channels_first" else -1
      def build(self, input_shape):
          self.dconv = DepthwiseConv2D((1, 1), depth_multiplier=1, use_bias=False)
      def call(self, inputs, **kwargs):
          pos = K.relu(inputs)
          neg = -1 * self.dconv(K.relu(-1 * inputs))
          return pos + neg
      def compute_output_shape(self, input_shape):
          return input_shape
      def get_config(self):
          config = super(My_PRELU_act, self).get_config()
          return config
      @classmethod
      def from_config(cls, config):
          return cls(**config)

  def convert_prelu_layer(layer):
      print(layer.name)
      if isinstance(layer, keras.layers.PReLU):
          print(">>>> Convert PReLu <<<<")
          return My_PRELU_act()
      return layer

  mm = keras.models.load_model('checkpoints/keras_se_mobile_facenet_emore_IV_basic_agedb_30_epoch_48_0.957833.h5', compile=False)
  mmn = keras.models.clone_model(mm, clone_function=convert_prelu_layer)
  ```
***

# OCR
  ```sh
  docker run -it -p 8866:8866 paddleocr:cpu bash
  git clone https://gitee.com/PaddlePaddle/PaddleOCR
  sed -i 's/ch_det_mv3_db/ch_det_r50_vd_db/' deploy/hubserving/ocr_system/params.py
  sed -i 's/ch_rec_mv3_crnn/ch_rec_r34_vd_crnn_enhance/' deploy/hubserving/ocr_system/params.py
  export PYTHONPATH=. && hub uninstall ocr_system; hub install deploy/hubserving/ocr_system/ && hub serving start -m ocr_system

  OCR_DID=`docker ps -a | sed -n '2,2p' | cut -d ' ' -f 1`
  docker cp ch_det_r50_vd_db_infer/* $OCR_DID:/PaddleOCR/inference/
  docker cp ch_rec_r34_vd_crnn_enhance_infer/* $OCR_DID:/PaddleOCR/inference/

  ```
  ```sh
  IMG_STR=`base64 -w 0 $TEST_PIC`
  echo "{\"images\": [\"`base64 -w 0 Selection_261.png`\"]}" > foo
  curl -H "Content-Type:application/json" -X POST --data "{\"images\": [\"填入图片Base64编码(需要删除'data:image/jpg;base64,'）\"]}" http://localhost:8866/predict/ocr_system
  curl -H "Content-Type:application/json" -X POST --data "{\"images\": [\"`base64 -w 0 Selection_101.png`\"]}" http://localhost:8866/predict/ocr_system
  curl -H "Content-Type:application/json" -X POST --data foo http://localhost:8866/predict/ocr_system
  echo "{\"images\": [\"`base64 -w 0 Selection_261.png`\"]}" | curl -v -X PUT -H 'Content-Type:application/json' -d @- http://localhost:8866/predict/ocr_system
  ```
  ```py
  import requests
  import base64
  import json
  from matplotlib.font_manager import FontProperties

  class PaddleOCR:
      def __init__(self, url, font='/usr/share/fonts/opentype/noto/NotoSerifCJK-Light.ttc'):
          self.url = url
          self.font = FontProperties(fname=font)
      def __call__(self, img_path, thresh=0.9, show=2):
          with open(img_path, 'rb') as ff:
              aa = ff.read()
          bb = base64.b64encode(aa).decode()
          rr = requests.post(self.url, headers={"Content-type": "application/json"}, data='{"images": ["%s"]}' % bb)
          dd = json.loads(rr.text)

          imm = imread(img_path)
          if show == 0:
              return dd
          if show == 1:
              fig, axes = plt.subplots(1, 1, sharex=True, sharey=True)
              axes = [axes, axes]
          elif show == 2:
              fig, axes = plt.subplots(1, 2, sharex=True, sharey=True)
              axes[1].invert_yaxis()
          axes[0].imshow(imm)
          for ii in dd['results'][0]:
              jj = np.array(ii['text_region'])
              kk = np.vstack([jj, jj[0]])
              axes[0].plot([ii[0] for ii in kk], [ii[1] for ii in kk], 'r')
              axes[1].text(jj[-1, 0], jj[-1, 1], ii['text'], fontproperties=font, fontsize=(jj[-1, 1]-jj[0, 1])/2)
          # plt.tight_layout()
          return dd

  pp = PaddleOCR("http://localhost:8866/predict/ocr_system")
  pp("./Selection_261.png")
  ```
  ```sh
  python3 tools/infer/predict_system.py --image_dir="./doc/imgs/2.jpg" --det_model_dir="./inference/det_db/"  --rec_model_dir="./inference/rec_crnn/"
  ```
  ```py
  # /opt/anaconda3/lib/python3.7/site-packages/onnx2keras/elementwise_layers.py
  def convert_reciprocal(node, params, layers, lambda_func, node_name, keras_name):
      """
      Convert element-wise division
      :param node: current operation node
      :param params: operation attributes
      :param layers: available keras layers
      :param lambda_func: function for keras Lambda layer
      :param node_name: internal converter name
      :param keras_name: resulting layer name
      :return: None
      """     
      logger = logging.getLogger('onnx2keras:reciprocal')
      print(layers[node.input[0]])

      if len(node.input) != 1:
          assert AttributeError('Not 1 input for reciprocal layer.')

      layers[node_name] = 1 / layers[node.input[0]]
  ```
  ```py
  # /opt/anaconda3/lib/python3.7/site-packages/onnx2keras/upsampling_layers.py
  def convert_resize(node, params, layers, lambda_func, node_name, keras_name):
      """
      Convert upsample.
      :param node: current operation node
      :param params: operation attributes
      :param layers: available keras layers
      :param lambda_func: function for keras Lambda layer
      :param node_name: internal converter name
      :param keras_name: resulting layer name
      :return: None
      """
      logger = logging.getLogger('onnx2keras:resize')
      logger.warning('!!! EXPERIMENTAL SUPPORT (resize) !!!')
      print([layers[ii] for ii in node.input])

      if len(node.input) != 1:
          if node.input[-1] in layers and isinstance(layers[node.input[-1]], np.ndarray):
              params['scales'] = layers[node.input[-1]]
          else:
              raise AttributeError('Unsupported number of inputs')

      if params['mode'].decode('utf-8') != 'nearest':
          logger.error('Cannot convert non-nearest upsampling.')
          raise AssertionError('Cannot convert non-nearest upsampling')

      scale = np.uint8(params['scales'][-2:])

      upsampling = keras.layers.UpSampling2D(
          size=scale, name=keras_name
      )   

      layers[node_name] = upsampling(layers[node.input[0]])
  ```
  ```py
  # /opt/anaconda3/lib/python3.7/site-packages/onnx2keras/operation_layers.py
  def convert_clip(node, params, layers, lambda_func, node_name, keras_name):
      """
      Convert clip layer
      :param node: current operation node
      :param params: operation attributes
      :param layers: available keras layers
      :param lambda_func: function for keras Lambda layer
      :param node_name: internal converter name
      :param keras_name: resulting layer name
      :return: None
      """
      logger = logging.getLogger('onnx2keras:clip')
      if len(node.input) != 1:
          assert AttributeError('More than 1 input for clip layer.')

      input_0 = ensure_tf_type(layers[node.input[0]], name="%s_const" % keras_name)
      print(node.input, [layers[ii] for ii in node.input], node_name, params)
      if len(node.input == 3):
          params['min'] = layers[node.input[1]]
          params['max'] = layers[node.input[2]]

      if params['min'] == 0:
          logger.debug("Using ReLU({0}) instead of clip".format(params['max']))
          layer = keras.layers.ReLU(max_value=params['max'], name=keras_name)
      else:
          def target_layer(x, vmin=params['min'], vmax=params['max']):
              import tensorflow as tf
              return tf.clip_by_value(x, vmin, vmax)
          layer = keras.layers.Lambda(target_layer, name=keras_name)
          lambda_func[keras_name] = target_layer

      layers[node_name] = layer(input_0)
  ```
  ```py
  # /opt/anaconda3/lib/python3.7/site-packages/onnx2keras/activation_layers.py
  def convert_hard_sigmoid(node, params, layers, lambda_func, node_name, keras_name):
      """
      Convert Sigmoid activation layer
      :param node: current operation node
      :param params: operation attributes
      :param layers: available keras layers
      :param lambda_func: function for keras Lambda layer
      :param node_name: internal converter name
      :param keras_name: resulting layer name
      :return: None
      """
      if len(node.input) != 1:
          assert AttributeError('More than 1 input for an activation layer.')

      input_0 = ensure_tf_type(layers[node.input[0]], name="%s_const" % keras_name)
      hard_sigmoid = keras.layers.Activation(keras.activations.hard_sigmoid, name=keras_name)
      layers[node_name] = hard_sigmoid(input_0)
  ```
  ```py
  from .activation_layers import convert_hard_sigmoid
  from .elementwise_layers import convert_reciprocal
  from .upsampling_layers import convert_resize
  ```
***

## Multi GPU
  ```py
  tf.debugging.set_log_device_placement(True)

  strategy = tf.distribute.MirroredStrategy()
  with strategy.scope():
    inputs = tf.keras.layers.Input(shape=(1,))
    predictions = tf.keras.layers.Dense(1)(inputs)
    model = tf.keras.models.Model(inputs=inputs, outputs=predictions)
    model.compile(loss='mse', optimizer=tf.keras.optimizers.SGD(learning_rate=0.2))

  dataset = tf.data.Dataset.from_tensors(([1.], [1.])).repeat(100).batch(10)
  model.fit(dataset, epochs=2)
  model.evaluate(dataset)
  ```
  ```py
  mirrored_strategy = tf.distribute.MirroredStrategy()
  # Compute global batch size using number of replicas.
  BATCH_SIZE_PER_REPLICA = 5
  global_batch_size = (BATCH_SIZE_PER_REPLICA *
                       mirrored_strategy.num_replicas_in_sync)
  dataset = tf.data.Dataset.from_tensors(([1.], [1.])).repeat(100)
  dataset = dataset.batch(global_batch_size)

  LEARNING_RATES_BY_BATCH_SIZE = {5: 0.1, 10: 0.15}
  learning_rate = LEARNING_RATES_BY_BATCH_SIZE[global_batch_size]

  with mirrored_strategy.scope():
    model = tf.keras.Sequential([tf.keras.layers.Dense(1, input_shape=(1,))])
    optimizer = tf.keras.optimizers.SGD()

  dataset = tf.data.Dataset.from_tensors(([1.], [1.])).repeat(1000).batch(
      global_batch_size)
  dist_dataset = mirrored_strategy.experimental_distribute_dataset(dataset)

  @tf.function
  def train_step(dist_inputs):
    def step_fn(inputs):
      features, labels = inputs

      with tf.GradientTape() as tape:
        # training=True is only needed if there are layers with different
        # behavior during training versus inference (e.g. Dropout).
        logits = model(features, training=True)
        cross_entropy = tf.nn.softmax_cross_entropy_with_logits(
            logits=logits, labels=labels)
        loss = tf.reduce_sum(cross_entropy) * (1.0 / global_batch_size)

      grads = tape.gradient(loss, model.trainable_variables)
      optimizer.apply_gradients(list(zip(grads, model.trainable_variables)))
      return cross_entropy

    per_example_losses = mirrored_strategy.experimental_run_v2(step_fn, args=(dist_inputs,))
    mean_loss = mirrored_strategy.reduce(
        tf.distribute.ReduceOp.MEAN, per_example_losses, axis=0)
    return mean_loss

  with mirrored_strategy.scope():
    for inputs in dist_dataset:
      print(train_step(inputs))
  ```
  ```py
  @tf.function
  def step_fn(inputs):
      return ss.experimental_assign_to_logical_device(mm.predict(inputs), 0)

  with ss.scope():
    ss.run(step_fn, args=(np.ones([2, 112, 112, 3]),))
  ```
## NPU
  ```sh
  资料：建议用谷歌浏览器或QQ浏览器浏览可以直接网页翻译！
  文档汇总： https://docs.khadas.com/vim3/index.html
  烧录教程： https://docs.khadas.com/vim3/UpgradeViaUSBCable.html
  硬件资料： https://docs.khadas.com/vim3/HardwareDocs.html
  安卓固件下载： https://docs.khadas.com/vim3/FirmwareAndroid.html
  Ubuntu固件下载： https://docs.khadas.com/vim3/FirmwareUbuntu.html
  第三方操作系统： https://docs.khadas.com/vim3/FirmwareThirdparty.html#AndroidTV

  VIM3释放 NPU资料流程: https://www.khadas.com/npu-toolkit-vim3
  ```
## 阿里云数据整理
  ```py
  import cv2
  import shutil
  import glob2
  from tqdm import tqdm
  from skimage.transform import SimilarityTransform
  from sklearn.preprocessing import normalize

  def face_align_landmarks(img, landmarks, image_size=(112, 112)):
      ret = []
      for landmark in landmarks:
          src = np.array(
              [[38.2946, 51.6963], [73.5318, 51.5014], [56.0252, 71.7366], [41.5493, 92.3655], [70.729904, 92.2041]],
              dtype=np.float32,
          )

          if image_size[0] != 112:
              src *= image_size[0] / 112
              src[:, 0] += 8.0

          dst = landmark.astype(np.float32)
          tform = SimilarityTransform()
          tform.estimate(dst, src)
          M = tform.params[0:2, :]
          ret.append(cv2.warpAffine(img, M, (image_size[1], image_size[0]), borderValue=0.0))

      return np.array(ret)

  def extract_face_images(source_reg, dest_path, detector, limit=-1):
      aa = glob2.glob(source_reg)
      dest_single = dest_path
      dest_multi = dest_single + '_multi'
      dest_none = dest_single + '_none'
      os.makedirs(dest_none, exist_ok=True)
      if limit != -1:
          aa = aa[:limit]
      for ii in tqdm(aa):
          imm = imread(ii)
          bbs, pps = detector(imm)
          if len(bbs) == 0:
              shutil.copy(ii, os.path.join(dest_none, '_'.join(ii.split('/')[-2:])))
              continue
          user_name, image_name = ii.split('/')[-2], ii.split('/')[-1]
          if len(bbs) == 1:
              dest_path = os.path.join(dest_single, user_name)
          else:
              dest_path = os.path.join(dest_multi, user_name)

          if not os.path.exists(dest_path):
              os.makedirs(dest_path)
          # if len(bbs) != 1:
          #     shutil.copy(ii, dest_path)

          nns = face_align_landmarks(imm, pps)
          image_name_form = '%s_{}.%s' % tuple(image_name.split('.'))
          for id, nn in enumerate(nns):
              dest_name = os.path.join(dest_path, image_name_form.format(id))
              imsave(dest_name, nn)

  import insightface
  os.environ['MXNET_CUDNN_AUTOTUNE_DEFAULT'] = '0'
  # retina = insightface.model_zoo.face_detection.retinaface_mnet025_v1()
  retina = insightface.model_zoo.face_detection.retinaface_r50_v1()
  retina.prepare(0)
  detector = lambda imm: retina.detect(imm)

  sys.path.append('/home/leondgarse/workspace/samba/tdFace-flask')
  from face_model.face_model import FaceModel
  det = FaceModel(None)

  def detector(imm):
      bbox, confid, points  = det.get_face_location(imm)
      return bbox, points
  extract_face_images("./face_image/*/*.jpg", 'tdevals', detector)
  extract_face_images("./tdface_Register/*/*.jpg", 'tdface_Register_cropped', detector)
  extract_face_images("./tdface_Register/*/*.jpg", 'tdface_Register_mtcnn', detector)

  ''' Review _multi and _none folder by hand, then do detection again on _none folder using another detector '''
  inn = glob2.glob('tdface_Register_mtcnn_none/*.jpg')
  for ii in tqdm(inn):
      imm = imread(ii)
      bbs, pps = detector(imm)
      if len(bbs) != 0:
          image_name = os.path.basename(ii)
          user_name = image_name.split('_')[0]
          dest_path = os.path.join(os.path.dirname(ii), user_name)
          os.makedirs(dest_path, exist_ok=True)
          nns = face_align_landmarks(imm, pps)
          image_name_form = '%s_{}.%s' % tuple(image_name.split('.'))
          for id, nn in enumerate(nns):
              dest_name = os.path.join(dest_path, image_name_form.format(id))
              imsave(dest_name, nn)
              os.rename(ii, os.path.join(dest_path, image_name))

  print(">>>> 提取特征值")
  model_path = "/home/tdtest/workspace/Keras_insightface/checkpoints/keras_resnet101_emore_II_triplet_basic_agedb_30_epoch_107_0.971000.h5"
  model = tf.keras.models.load_model(model_path, compile=False)
  interp = lambda ii: normalize(model.predict((np.array(ii) - 127.5) / 127))
  register_path = 'tdface_Register_mtcnn'

  backup_file = 'tdface_Register_mtcnn.npy'
  if os.path.exists(backup_file):
      imms = np.load('tdface_Register_mtcnn.npy')
  else:
      imms = glob2.glob(os.path.join(register_path, "*/*.jpg"))
      np.save('tdface_Register_mtcnn.npy', imms)

  batch_size = 64
  steps = int(np.ceil(len(imms) / batch_size))
  embs = []
  for ii in tqdm(range(steps), total=steps):
      ibb = imms[ii * batch_size : (ii + 1) * batch_size]
      embs.append(interp([imread(jj) for jj in ibb]))

  embs = np.concatenate(embs)
  dd, pp = {}, {}
  for ii, ee in zip(imms, embs):
      user = os.path.basename(os.path.dirname(ii))
      dd[user] = np.vstack([dd.get(user, np.array([]).reshape(0, embs.shape[-1])), [ee]])
      pp[user] = np.hstack([pp.get(user, []), ii])
  # dd_bak = dd.copy()
  # pp_bak = pp.copy()
  print("Total: %d" % (len(dd)))

  print(">>>> 合并组间距离过小的成员")
  OUT_THREASH = 0.7
  tt = dd.copy()
  while len(tt) > 0:
      kk, vv = tt.popitem()
      # oo = np.vstack(list(tt.values()))
      for ikk, ivv in tt.items():
          imax = np.dot(vv, ivv.T).max()
          if imax > OUT_THREASH:
              print("First: %s, Second: %s, Max dist: %.4f" % (kk, ikk, imax))
              if kk in dd and ikk in dd:
                  dd[kk] = np.vstack([dd[kk], dd[ikk]])
                  dd.pop(ikk)
                  pp[kk] = np.hstack([pp[kk], pp[ikk]])
                  pp.pop(ikk)
  # print([kk for kk, vv in pp.items() if vv.shape[0] != dd[kk].shape[0]])
  print("Total left: %d" % (len(dd)))

  ''' Similar images between users '''
  src = 'tdface_Register_mtcnn'
  dst = 'tdface_Register_mtcnn_simi'
  with open('tdface_Register_mtcnn.foo', 'r') as ff:
      aa = ff.readlines()
  for id, ii in tqdm(enumerate(aa), total=len(aa)):
      first, second, simi = [jj.split(': ')[1] for jj in ii.strip().split(', ')]
      dest_path = os.path.join(dst, '_'.join([str(id), first, second, simi]))
      os.makedirs(dest_path, exist_ok=True)
      for pp in os.listdir(os.path.join(src, first)):
          src_path = os.path.join(src, first, pp)
          shutil.copy(src_path, os.path.join(dest_path, first + '_' + pp))
      for pp in os.listdir(os.path.join(src, second)):
          src_path = os.path.join(src, second, pp)
          shutil.copy(src_path, os.path.join(dest_path, second + '_' + pp))

  ''' Pos & Neg dists '''
  batch_size = 128
  gg = ImageDataGenerator(rescale=1./255, preprocessing_function=lambda img: (img - 0.5) * 2)
  tt = gg.flow_from_directory('./tdevals', target_size=(112, 112), batch_size=batch_size)
  steps = int(np.ceil(tt.classes.shape[0] / batch_size))
  embs = []
  classes = []
  for _ in tqdm(range(steps), total=steps):
      aa, bb = tt.next()
      emb = interp(aa)
      embs.extend(emb)
      classes.extend(np.argmax(bb, 1))
  embs = np.array(embs)
  classes = np.array(classes)
  class_matrix = np.equal(np.expand_dims(classes, 0), np.expand_dims(classes, 1))
  dists = np.dot(embs, embs.T)
  pos_dists = np.where(class_matrix, dists, np.ones_like(dists))
  neg_dists = np.where(np.logical_not(class_matrix), dists, np.zeros_like(dists))
  (neg_dists.max(1) <= pos_dists.min(1)).sum()
  ```
  ```py
  import glob2
  def dest_test(aa, bb, model, reg_path="./tdface_Register_mtcnn"):
      ees = []
      for ii in [aa, bb]:
          iee = glob2.glob(os.path.join(reg_path, ii, "*.jpg"))
          iee = [imread(jj) for jj in iee]
          ees.append(normalize(model.predict((np.array(iee) / 255. - 0.5) * 2)))
      return np.dot(ees[0], ees[1].T)

  with open('tdface_Register_mtcnn_0.7.foo', 'r') as ff:
      aa = ff.readlines()
  for id, ii in enumerate(aa):
      first, second, simi = [jj.split(': ')[1] for jj in ii.strip().split(', ')]
      dist = dest_test(first, second, model)
      if dist < OUT_THREASH:
          print(("first = %s, second = %s, simi = %s, model_simi = %f" % (first, second, simi, dist)))
  ```
  ```sh
  cp ./face_image/2818/1586472495016.jpg ./face_image/4609/1586475252234.jpg ./face_image/3820/1586472950858.jpg ./face_image/4179/1586520054080.jpg ./face_image/2618/1586471583221.jpg ./face_image/6696/1586529149923.jpg ./face_image/5986/1586504872276.jpg ./face_image/1951/1586489518568.jpg ./face_image/17/1586511238696.jpg ./face_image/17/1586511110105.jpg ./face_image/17/1586511248992.jpg ./face_image/4233/1586482466485.jpg ./face_image/5500/1586493019872.jpg ./face_image/4884/1586474119164.jpg ./face_image/5932/1586471784905.jpg ./face_image/7107/1586575911740.jpg ./face_image/4221/1586512334133.jpg ./face_image/5395/1586578437529.jpg ./face_image/4204/1586506059923.jpg ./face_image/4053/1586477985553.jpg ./face_image/7168/1586579239307.jpg ./face_image/7168/1586489559660.jpg ./face_image/5477/1586512847480.jpg ./face_image/4912/1586489637333.jpg ./face_image/5551/1586502762688.jpg ./face_image/5928/1586579219121.jpg ./face_image/6388/1586513897953.jpg ./face_image/4992/1586471873460.jpg ./face_image/5934/1586492793214.jpg ./face_image/5983/1586490703112.jpg ./face_image/5219/1586492929098.jpg ./face_image/5203/1586487204198.jpg ./face_image/6099/1586490074263.jpg ./face_image/5557/1586490232722.jpg ./face_image/4067/1586491778846.jpg ./face_image/4156/1586512886040.jpg ./face_image/5935/1586492829221.jpg ./face_image/2735/1586513495061.jpg ./face_image/5264/1586557233625.jpg ./face_image/1770/1586470942329.jpg ./face_image/7084/1586514100804.jpg ./face_image/5833/1586497276529.jpg ./face_image/2200/1586577699180.jpg tdevals_none
  ```
## tf dataset cache
  - **dataset.cache** **MUST** be placed **before** data random augment and shuffle
  ```py
  dd = np.arange(30).reshape(3, 10)

  ''' Cache before shuffle and random '''
  ds = tf.data.Dataset.from_tensor_slices(dd)
  # ds = ds.cache()
  ds = ds.shuffle(dd.shape[0])
  ds = ds.map(lambda xx: xx + tf.random.uniform((1,), 1, 10, dtype=tf.int64))

  for ii in range(3):
      print(">>>> Epoch:", ii)
      for jj in ds:
          print(jj)
  # >>>> Epoch: 0
  # tf.Tensor([ 9 10 11 12 13 14 15 16 17 18], shape=(10,), dtype=int64)
  # tf.Tensor([13 14 15 16 17 18 19 20 21 22], shape=(10,), dtype=int64)
  # tf.Tensor([23 24 25 26 27 28 29 30 31 32], shape=(10,), dtype=int64)
  # >>>> Epoch: 1
  # tf.Tensor([11 12 13 14 15 16 17 18 19 20], shape=(10,), dtype=int64)
  # tf.Tensor([21 22 23 24 25 26 27 28 29 30], shape=(10,), dtype=int64)
  # tf.Tensor([ 9 10 11 12 13 14 15 16 17 18], shape=(10,), dtype=int64)
  # >>>> Epoch: 2
  # tf.Tensor([23 24 25 26 27 28 29 30 31 32], shape=(10,), dtype=int64)
  # tf.Tensor([12 13 14 15 16 17 18 19 20 21], shape=(10,), dtype=int64)
  # tf.Tensor([ 3  4  5  6  7  8  9 10 11 12], shape=(10,), dtype=int64)

  ''' Cache before random but after shuffle '''
  ds2 = tf.data.Dataset.from_tensor_slices(dd)
  ds2 = ds2.shuffle(dd.shape[0])
  ds2 = ds2.cache()
  ds2 = ds2.map(lambda xx: xx + tf.random.uniform((1,), 1, 10, dtype=tf.int64))

  for ii in range(3):
      print(">>>> Epoch:", ii)
      for jj in ds2:
          print(jj)
  # >>>> Epoch: 0
  # tf.Tensor([26 27 28 29 30 31 32 33 34 35], shape=(10,), dtype=int64)
  # tf.Tensor([17 18 19 20 21 22 23 24 25 26], shape=(10,), dtype=int64)
  # tf.Tensor([ 6  7  8  9 10 11 12 13 14 15], shape=(10,), dtype=int64)
  # >>>> Epoch: 1
  # tf.Tensor([22 23 24 25 26 27 28 29 30 31], shape=(10,), dtype=int64)
  # tf.Tensor([17 18 19 20 21 22 23 24 25 26], shape=(10,), dtype=int64)
  # tf.Tensor([ 3  4  5  6  7  8  9 10 11 12], shape=(10,), dtype=int64)
  # >>>> Epoch: 2
  # tf.Tensor([21 22 23 24 25 26 27 28 29 30], shape=(10,), dtype=int64)
  # tf.Tensor([15 16 17 18 19 20 21 22 23 24], shape=(10,), dtype=int64)
  # tf.Tensor([ 3  4  5  6  7  8  9 10 11 12], shape=(10,), dtype=int64)

  ''' Cache after random and shuffle '''
  ds3 = tf.data.Dataset.from_tensor_slices(dd)
  ds3 = ds3.shuffle(dd.shape[0])
  ds3 = ds3.map(lambda xx: xx + tf.random.uniform((1,), 1, 10, dtype=tf.int64))
  ds3 = ds3.cache()

  for ii in range(3):
      print(">>>> Epoch:", ii)
      for jj in ds3:
          print(jj)
  # >>>> Epoch: 0
  # tf.Tensor([24 25 26 27 28 29 30 31 32 33], shape=(10,), dtype=int64)
  # tf.Tensor([14 15 16 17 18 19 20 21 22 23], shape=(10,), dtype=int64)
  # tf.Tensor([ 4  5  6  7  8  9 10 11 12 13], shape=(10,), dtype=int64)
  # >>>> Epoch: 1
  # tf.Tensor([24 25 26 27 28 29 30 31 32 33], shape=(10,), dtype=int64)
  # tf.Tensor([14 15 16 17 18 19 20 21 22 23], shape=(10,), dtype=int64)
  # tf.Tensor([ 4  5  6  7  8  9 10 11 12 13], shape=(10,), dtype=int64)
  # >>>> Epoch: 2
  # tf.Tensor([24 25 26 27 28 29 30 31 32 33], shape=(10,), dtype=int64)
  # tf.Tensor([14 15 16 17 18 19 20 21 22 23], shape=(10,), dtype=int64)
  # tf.Tensor([ 4  5  6  7  8  9 10 11 12 13], shape=(10,), dtype=int64)
  ```
## ReLU Activation
  ```py
  keras.activations.relu 将矩阵x内所有负值都设为零，其余的值不变
  `max(x, 0)`

  keras.activations.elu
  `x` if `x > 0`
  `alpha * (exp(x)-1)` if `x < 0`

  keras.layers.LeakyReLU
  `f(x) = alpha * x for x < 0`
  `f(x) = x for x >= 0`


  keras.layers.PReLU
  `f(x) = alpha * x for x < 0`
  `f(x) = x for x >= 0`
  where `alpha` is a learned array with the same shape as x
  ```
## Route trace
  traceroute www.baidu.com
  dig +trace www.baidu.com
  speedtest-cli
  mtr -r -c 30 -s 1024 www.baidu.com

  !wget http://im.tdweilai.com:38831/keras_ResNest101_emore_II_basic_agedb_30_epoch_64_0.968500.h5

  netstat -na | awk -F ' ' 'BEGIN {WAIT_C=0; EST_C=0} /:6379 /{if($NF == "TIME_WAIT"){WAIT_C=WAIT_C+1}else{EST_C=EST_C+1}} END {print "TIME_WAIT: "WAIT_C", ESTABLISHED: "EST_C}'

  watch -tn 3 'echo ">>>> php connection:"; ss -pl | grep php | wc -l; echo ">>>> 8800 Chat-Server connection:"; netstat -na | grep -i ":8800 " | wc -l; echo ">>>> 8812 Fs-Server connection:"; netstat -na | grep -i ":8812 " | wc -l; echo ">>>> 6379 redis-server connection:"; netstat -na | grep -i ":6379 " | wc -l; echo ">>>> Socket status:"; ss -s; echo ">>>> top"; top -b -n 1'

  查看占用端口的进程 ss -lntpd | grep :4444

  ss -lntpd | grep :8800 -- pid=8431
  ss -lntpd | grep :8812 -- pid=9029

  ps -lax | grep 8431 -- ppid 470
  ps -lax | grep 9029 -- ppid 470

  docker container ls

  3306 docker-containerd -- 470 docker-containerd-shim tdwl_ws -- 8431 Chat-Server:master -- 8432 Chat-Server:manager -- MULTI Chat-Server:work
                                                               -- 9029 Fs-Server:master -- 9030 Fs-Server:manager -- MULTI Fs-Server:work
                         -- 23830 docker-containerd-shim fs -- 24107 freeswitch -nonat -nc
                         -- 23847 bash -- 24153 fs_cli

  1 -- 1637 /usr/bin/dockerd -- 3306 docker-containerd

  ss -lntpd | grep -i freeswitch
  ss -lntpd | grep -i php
  ss -lntpd | grep -i redis
***
