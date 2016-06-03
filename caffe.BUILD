load("@//third_party:caffe.bzl", "if_cuda")
load("@tf//tensorflow/core:platform/default/build_config.bzl",
     "tf_get_cuda_version",
     "tf_get_cudnn_version",
    )

package(default_visibility = ["//visibility:public"])

CAFFE_LAYERS_OBJS = [
    "threshold_layer.cpp.o",
    "tile_layer.cpp.o",
    "window_data_layer.cpp.o",
    "absval_layer.cpp.o",
    "accuracy_layer.cpp.o",
    "argmax_layer.cpp.o",
    "base_conv_layer.cpp.o",
    "base_data_layer.cpp.o",
    "batch_norm_layer.cpp.o",
    "batch_reindex_layer.cpp.o",
    "bias_layer.cpp.o",
    "bnll_layer.cpp.o",
    "concat_layer.cpp.o",
    "contrastive_loss_layer.cpp.o",
    "conv_layer.cpp.o",
    "crop_layer.cpp.o",
    "cudnn_conv_layer.cpp.o",
    "cudnn_lcn_layer.cpp.o",
    "cudnn_lrn_layer.cpp.o",
    "cudnn_pooling_layer.cpp.o",
    "cudnn_relu_layer.cpp.o",
    "cudnn_sigmoid_layer.cpp.o",
    "cudnn_softmax_layer.cpp.o",
    "cudnn_tanh_layer.cpp.o",
    "data_layer.cpp.o",
    "deconv_layer.cpp.o",
    "dropout_layer.cpp.o",
    "dummy_data_layer.cpp.o",
    "eltwise_layer.cpp.o",
    "elu_layer.cpp.o",
    "embed_layer.cpp.o",
    "euclidean_loss_layer.cpp.o",
    "exp_layer.cpp.o",
    "filter_layer.cpp.o",
    "flatten_layer.cpp.o",
    "hdf5_data_layer.cpp.o",
    "hdf5_output_layer.cpp.o",
    "hinge_loss_layer.cpp.o",
    "im2col_layer.cpp.o",
    "image_data_layer.cpp.o",
    "infogain_loss_layer.cpp.o",
    "inner_product_layer.cpp.o",
    "input_layer.cpp.o",
    "log_layer.cpp.o",
    "loss_layer.cpp.o",
    "lrn_layer.cpp.o",
    "memory_data_layer.cpp.o",
    "multinomial_logistic_loss_layer.cpp.o",
    "mvn_layer.cpp.o",
    "neuron_layer.cpp.o",
    "parameter_layer.cpp.o",
    "pooling_layer.cpp.o",
    "power_layer.cpp.o",
    "prelu_layer.cpp.o",
    "reduction_layer.cpp.o",
    "relu_layer.cpp.o",
    "reshape_layer.cpp.o",
    "scale_layer.cpp.o",
    "sigmoid_cross_entropy_loss_layer.cpp.o",
    "sigmoid_layer.cpp.o",
    "silence_layer.cpp.o",
    "slice_layer.cpp.o",
    "softmax_layer.cpp.o",
    "softmax_loss_layer.cpp.o",
    "split_layer.cpp.o",
    "spp_layer.cpp.o",
    "tanh_layer.cpp.o",
    "layer_factory.cpp.o",
]

genrule(
    name = "configure",
    srcs = if_cuda([
        "@tf//third_party/gpus/cuda:include/cudnn.h",
        "@tf//third_party/gpus/cuda:lib64/libcudnn.so" + tf_get_cudnn_version()
    ]),
    message = "Building Caffe (this may take a while)",
    outs = [
        "lib/libcaffe.a", 
        "lib/libproto.a", 
        "include/caffe/proto/caffe.pb.h"
    ],
    cmd = '''
        srcdir=$$(pwd);
        workdir=$$(mktemp -d -t tmp.XXXXXXXXXX); ''' + 
        if_cuda(''' 
            cudnn_includes=$(location @tf//third_party/gpus/cuda:include/cudnn.h);
            cudnn_lib=$(location @tf//third_party/gpus/cuda:lib64/libcudnn.so%s);
            extra_cmake_opts="-DCPU_ONLY:bool=OFF
                              -DUSE_CUDNN:bool=ON 
                              -DCUDNN_INCLUDE:path=$$srcdir/$$(dirname $$cudnn_includes)
                              -DCUDNN_LIBRARY:path=$$srcdir/$$cudnn_lib"; ''' % tf_get_cudnn_version(), 
            '''extra_cmake_opts="-DCPU_ONLY:bool=ON";''') +
        '''
        pushd $$workdir;
        cmake $$srcdir/external/caffe_git         \
            -DCMAKE_INSTALL_PREFIX=$$srcdir/$(@D) \
            -DCMAKE_BUILD_TYPE=Release            \
            -DBLAS:string="open"                  \
            -DBUILD_python=OFF                    \
            -DBUILD_python_layer=OFF              \
            -DUSE_OPENCV=OFF                      \
            -DBUILD_SHARED_LIBS=OFF               \
            $${extra_cmake_opts};
        cmake --build . -- -j 4;
        cmake --build . --target install;
        popd;
        rm -rf $$workdir;''',
)

genrule(
    name = "cuda-extras",
    srcs = ["@tf//third_party/gpus/cuda:cuda.config"],
    outs = ["lib64/libcurand.so" + tf_get_cuda_version()],
    cmd  = '''
        source $(location @tf//third_party/gpus/cuda:cuda.config) || exit -1;
        CUDA_TOOLKIT_PATH=$${CUDA_TOOLKIT_PATH:-/usr/local/cuda};
        FILE=libcurand.so%s;
        SRC=$$CUDA_TOOLKIT_PATH/lib64/$$FILE;

        if test ! -e $$SRC; then
            echo "ERROR: $$SRC cannot be found";
            exit -1;
        fi

        mkdir -p $(@D);
        cp $$SRC $(@D)/$$FILE;
    ''' % tf_get_cuda_version(),
) 

# TODO(rayg): Bazel will ignore `alwayslink=1` for *.a archives (a bug?). 
#   This genrule unpacks the caffe.a so the object files can be linked 
#   independantly. A terrible hack, not least because we
#   need to know the layer names upfront.
genrule(
    name = "caffe-extract",
    srcs = [":configure", "lib/libcaffe.a"],
    outs = ["libcaffe.a.dir/" + o for o in CAFFE_LAYERS_OBJS],
    cmd = '''
        workdir=$$(mktemp -d -t tmp.XXXXXXXXXX); 
        cp $(location :lib/libcaffe.a) $$workdir; 
        pushd $$workdir; 
        ar x libcaffe.a; 
        popd;
        mkdir -p $(@D)/libcaffe.a.dir;
        cp -a $$workdir/*.o $(@D)/libcaffe.a.dir; 
        rm -rf $$workdir;
        ''',
)

cc_library(
    name = "curand",
    srcs = [
        "lib64/libcurand.so" + tf_get_cuda_version(),
    ],
    data = [
        "lib64/libcurand.so" + tf_get_cuda_version()
    ],
    linkstatic = 1
)

cc_library(
    name = "caffe",
    srcs = [":caffe-extract", "lib/libcaffe.a", "lib/libproto.a"],
    hdrs = glob(["include/**"]) + ["include/caffe/proto/caffe.pb.h"],
    deps = if_cuda([
	"@tf//third_party/gpus/cuda:cudnn", 
	"@tf//third_party/gpus/cuda:cublas",
        ":curand",
    ]),
    data = if_cuda([
        "@tf//third_party/gpus/cuda:cudnn",
        "@tf//third_party/gpus/cuda:cublas",
        ":curand",
    ]),
    includes = ["include/"],
    defines = if_cuda([], ["CPU_ONLY"]),
    linkopts = [
        "-L/usr/lib/x86_64-linux-gnu/hdf5/serial/lib",
        "-lboost_system",
        "-lboost_thread",
        "-lboost_filesystem",
        "-lpthread",
        "-lglog",
        "-lgflags",
        "-lprotobuf",
        "-lhdf5_hl",
        "-lhdf5",
        "-lz",
        "-ldl",
        "-lm",
        "-llmdb",
        "-lleveldb",
        "-lsnappy",
        "-lopenblas",
        "-Wl,-rpath,/usr/local/lib:/usr/lib/x86_64-linux-gnu/hdf5/serial/lib",
    ],
    visibility = ["//visibility:public"],
    alwayslink = 1,
    linkstatic = 1,
)
